# အခန်း ၃ — Service Communication Landscape

## နိဒါန်း

Microservices architecture တွင် services တွင်းကနေ ကြားဖြတ်ဆက်သွယ်မှု (inter-service communication) သည် system ၏ reliability, performance, နှင့် scalability ကို တိုက်ရိုက် သက်ရောက်သော အရေးကြီးဆုံး design decision တစ်ခုဖြစ်သည်။ Monolith တွင် function call တစ်ကြိမ် ခေါ်ဆိုပြီး ပြီးဆုံးနေသော operation ကို Microservices တွင် network boundary ကျော်ကာ service များကြား ဆက်သွယ်ရသည်။ ဤ network hop (ကွန်ရက်ကူးဆင်းမှု) သည် latency, failure, partial failure တို့ဖြစ်ပေါ်နိုင်ချေကို ယူဆောင်လာသည်။ ဤအခန်းတွင် communication ပုံစံ သုံးမျိုး (Synchronous, Asynchronous, Real-Time) ကို နှိုင်းယှဉ်ကာ ဘယ်အချိန်တွင် ဘယ် protocol ကို ရွေးချယ်မည်ဆိုသည်ကို decision framework ဖြင့် ဆွေးနွေးသွားမည်ဖြစ်သည်။

---

## ၃.၁ Synchronous vs Asynchronous vs Real-Time

### Synchronous Communication (တပြိုင်နက် ဆက်သွယ်မှု)

**Synchronous** ဆိုသည်မှာ caller (ခေါ်သူ) သည် callee (ခေါ်ခံသူ) ၏ response ပြန်မရမချင်း wait (စောင့်) နေသော communication pattern ဖြစ်သည်။

```
Caller                    Callee
  │                         │
  │──── HTTP Request ───────▶│
  │         (blocking)       │
  │◀─── HTTP Response ───────│  ← response ရမှသာ ဆက်လက်သွားနိုင်သည်
  │                         │
```

**Protocols**: HTTP/REST, gRPC, GraphQL

**ဥပမာ** — User checkout လုပ်ချိန်

```python
# checkout_service.py

class CheckoutService:
    def checkout(self, cart_id: str, payment_info: dict) -> dict:
        # Step 1: Cart ကို ယူသည် (synchronous)
        cart = self.cart_client.get_cart(cart_id)  # wait လုပ်ရသည်

        # Step 2: Payment process လုပ်သည် (synchronous)
        payment = self.payment_client.charge(payment_info, cart.total)  # wait

        # Step 3: Order ဖန်တီးသည် (synchronous)
        order = self.order_client.create_order(cart, payment)  # wait

        return {"order_id": order.id, "status": "CONFIRMED"}
```

**Pros/Cons**
```
Pros:
├── Simple to implement and reason about
├── Immediate feedback
└── Strong consistency possible

Cons:
├── Temporal coupling — callee offline ဆိုလျှင် caller fail ဖြစ်သည်
├── Cascading failures ဖြစ်နိုင်ချေ
└── Response time = sum of all downstream calls
```

### Asynchronous Communication (အချိန်မတိုင်ပါ ဆက်သွယ်မှု)

**Asynchronous** ဆိုသည်မှာ caller သည် message (messages broker တွင်) publish လုပ်ပြီး response ကို စောင့်မနေဘဲ ဆက်လက် ဆောင်ရွက်နိုင်သည်။

```
Producer              Message Broker          Consumer
   │                       │                     │
   │── Publish Event ──────▶│                     │
   │  (non-blocking)        │──── Deliver ────────▶│
   │ (ဆက်လုပ်နိုင်သည်)        │                     │ (process လုပ်သည်)
```

**Protocols**: AMQP (RabbitMQ), Apache Kafka, AWS SQS/SNS

**ဥပမာ** — Order placed ပြီးနောက် notification

```python
# order_service.py

import json
import pika  # RabbitMQ client

class OrderService:
    def __init__(self, rabbitmq_channel):
        self.channel = rabbitmq_channel

    def place_order(self, order_data: dict) -> str:
        # Order ကို database တွင် save လုပ်သည်
        order_id = self.save_order(order_data)

        # Notification service ကို message publish လုပ်သည် (non-blocking)
        event = {
            "event_type": "ORDER_PLACED",
            "order_id": order_id,
            "customer_id": order_data["customer_id"],
            "total": str(order_data["total"])
        }
        self.channel.basic_publish(
            exchange="order_events",
            routing_key="order.placed",
            body=json.dumps(event)
        )
        # Notification service ၏ response ကို မစောင့်ဘဲ return
        return order_id
```

### Real-Time Communication

**Real-Time** ဆိုသည်မှာ server မှ client သို့ data ကို push လုပ်သည့် (သို့) bidirectional (နှစ်ဖက်) low-latency communication ဖြစ်သည်။

**Protocols**: WebSocket, Server-Sent Events (SSE), gRPC bidirectional streaming

```
Client                    Server
  │◀═══ WebSocket Upgrade ══▶│  (persistent connection)
  │                          │
  │◀──── Live Price Update ───│  (server push)
  │◀──── Live Price Update ───│
  │──── User Bid ────────────▶│
  │◀──── Bid Confirmed ───────│
```

---

## ၃.၂ Consistency, Availability, Coupling Triangle

Microservices communication တွင် **three-way trade-off** ရှိသည် —

```
              Consistency
              (ညီညွတ်မှု)
                   △
                  /│\
                 / │ \
                /  │  \
               /   │   \
              /    │    \
             /     │     \
            ▽──────┴──────▽
     Availability        Low Coupling
    (ရရှိနိုင်မှု)        (ပေါ်မီမှုနည်းမှု)
```

**Trade-off ဆုံးဖြတ်ချက်များ**

| ဆုံးဖြတ်ချက် | Consistency | Availability | Coupling | Communication |
|---|---|---|---|---|
| Sync + same transaction | မြင့် | နိမ့် | မြင့် | HTTP/gRPC |
| Sync + 2-phase commit | မြင့် | နိမ့် | မြင့် | Distributed TX |
| Async + eventual | နိမ့် (eventual) | မြင့် | နိမ့် | Message Queue |
| Async + outbox pattern | မြင့် (eventual) | မြင့် | နိမ့် | CDC + Events |

### CAP Theorem နှင့် Microservices

```
CAP (Consistency, Availability, Partition Tolerance)

Microservices ကွန်ရက်တွင် Partition (ကွဲကွာမှု) ဖြစ်နိုင်ချေ အမြဲရှိသည်
ထို့ကြောင့် CP သို့ AP ကိုသာ ရွေးချယ်နိုင်သည်

CP (Consistency + Partition Tolerance):
  → Strong consistency ပိုအရေးကြီးသောနေရာ
  → Bank transactions, inventory reservation

AP (Availability + Partition Tolerance):
  → Always available ပိုအရေးကြီးသောနေရာ
  → User profile reads, product catalog views
```

---

## ၃.၃ Protocol Selection per Use Case

Protocol မည်သည်ကို ရွေးမည်ဆိုသည်မှာ use case ပေါ် မူတည်သည်။

```
Use Case Matrix
──────────────────────────────────────────────────────────────────
Use Case              | Protocol      | Reason
──────────────────────────────────────────────────────────────────
User login (sync req) | REST/HTTP     | Simple request-response
Internal svc to svc   | gRPC          | Performance, type-safe
Real-time dashboard   | WebSocket/SSE | Server push
Order events          | Kafka         | High throughput, replay
Email notifications   | RabbitMQ      | Reliable delivery
File uploads          | REST + S3     | Large payloads
Mobile BFF            | GraphQL       | Flexible queries
Batch processing      | Kafka         | Stream processing
──────────────────────────────────────────────────────────────────
```

**Latency Requirements မူတည်သော ရွေးချယ်မှု**

```
Latency < 10ms   → gRPC (internal, same datacenter)
Latency < 100ms  → REST/HTTP/2 (internal/external)
Latency < 1s     → Async messaging (fire and forget)
Latency > 1s ok  → Batch/queue-based processing
```

---

## ၃.၄ Hybrid Architectures

Real-world system တွင် communication pattern တစ်ခုတည်းကိုသာ အသုံးပြုခြင်း မရှိပါ။ Synchronous နှင့် Asynchronous တို့ကို ပေါင်းစပ် (hybrid) သုံးကြသည်။

### Sync-then-Async Pattern

```
Client    API Gateway    OrderService    [Kafka]    NotificationService
  │           │               │             │              │
  │──POST ──▶ │               │             │              │
  │           │──CreateOrder─▶│             │              │
  │           │               │──Publish──▶ │              │
  │           │               │  "ORDER_    │              │
  │           │◀── 201 ───────│   PLACED"   │              │
  │◀── 201 ── │               │             │──Consume────▶│
  │  (return) │               │             │   & Send     │
  │           │               │             │    Email     │
```

**ရှင်းလင်းချက်** — Order create လုပ်ခြင်းသည် synchronous (user response ချက်ချင်းရသည်)။ Email ပို့ခြင်းသည် asynchronous (user ကို ကြားဖြတ်မဆိုင်)။

### Saga Pattern (Distributed Transactions)

```python
# saga_orchestrator.py

class OrderSaga:
    """
    Distributed transaction ကို Saga pattern ဖြင့် စီမံသည်
    Steps တစ်ခုမအောင်မြင်ပါက compensating transactions ဖြင့် undo လုပ်သည်
    """
    async def execute(self, order_data: dict):
        saga_steps = []
        try:
            # Step 1: Reserve inventory (sync)
            reservation = await self.inventory_svc.reserve(order_data["items"])
            saga_steps.append(("inventory", reservation.id))

            # Step 2: Process payment (sync)
            payment = await self.payment_svc.charge(order_data["payment"])
            saga_steps.append(("payment", payment.id))

            # Step 3: Create order (sync)
            order = await self.order_svc.create(order_data)
            saga_steps.append(("order", order.id))

            # Step 4: Notify (async — fire and forget)
            await self.notify_async(order.id)

            return order

        except Exception as e:
            # Compensating transactions — reverse order
            await self.compensate(saga_steps, e)
            raise

    async def compensate(self, completed_steps: list, error):
        # Completed steps ကို reverse order ဖြင့် undo လုပ်သည်
        for step_name, step_id in reversed(completed_steps):
            if step_name == "inventory":
                await self.inventory_svc.release_reservation(step_id)
            elif step_name == "payment":
                await self.payment_svc.refund(step_id)
```

### Event-Driven with CQRS

```
Command Side (Write):
Client → OrderService → Save to Write DB → Publish Event

Query Side (Read):
Client → QueryService → Read from Read DB (denormalized view)

Event Flow:
OrderService publishes "OrderPlaced"
   ↓
OrderReadModelUpdater (Subscribe) → Update Read DB
   ↓
Client queries OrderQueryService → Gets optimized read view
```

---

## ၃.၅ Communication Pattern Decision Tree

မည်သည့် communication pattern ကို ရွေးမည်ဆိုသည်ကို ဤ decision tree ဖြင့် ဦးတည်ပါ —

```
START: Service ကြား ဆက်သွယ်ရမည်
│
├─▶ Real-time data push လိုသလား? (chat, live updates)
│     ├── YES → WebSocket / SSE / gRPC Bidirectional Streaming
│     └── NO  → ▼
│
├─▶ Response ချက်ချင်း လိုသလား? (user request/reply cycle)
│     ├── YES → Synchronous Communication
│     │           ├── Internal service? → gRPC
│     │           └── External/Public?  → REST/HTTP
│     └── NO  → ▼
│
├─▶ High throughput, event streaming လိုသလား?
│     ├── YES → Apache Kafka / AWS Kinesis
│     └── NO  → ▼
│
├─▶ Reliable message delivery, work queues လိုသလား?
│     ├── YES → RabbitMQ / AWS SQS
│     └── NO  → ▼
│
└─▶ Simple notification, fan-out လိုသလား?
      ├── YES → AWS SNS / Redis Pub/Sub
      └── NO  → Requirements ကို ပြန်ဆန်းစစ်ပါ
```

**ဆုံးဖြတ်ချက် matrix အပြည့်အစုံ**

```
Pattern            | When to Use                    | Tool Examples
───────────────────────────────────────────────────────────────────────
REST/HTTP          | Public APIs, external clients  | FastAPI, Express
gRPC               | Internal high-perf services    | gRPC-Go, grpcio
GraphQL            | Flexible client-driven queries | Apollo, Strawberry
WebSocket          | Real-time bidirectional        | Socket.io, ws
SSE                | Server-to-client push (1-way)  | Native browser API
Kafka              | Event streaming, audit log     | Apache Kafka
RabbitMQ           | Task queues, RPC patterns      | AMQP
AWS SQS/SNS        | Cloud-managed messaging        | AWS SDK
Redis Pub/Sub      | Lightweight fan-out            | Redis
───────────────────────────────────────────────────────────────────────
```

**Resilience Patterns မည်သည်ကမဆို Synchronous Communication တွင် လိုအပ်သည်**

```python
# resilience_patterns.py
import asyncio
from circuitbreaker import circuit  # circuit breaker library

class ResilientServiceClient:

    @circuit(failure_threshold=5, recovery_timeout=30)
    async def call_with_circuit_breaker(self, endpoint: str, data: dict):
        """
        Circuit Breaker: 5 ကြိမ် fail ဖြစ်ပါက circuit open ဖြစ်သည်
        30 seconds ကြာမှ half-open ဖြစ်ကာ retry ဦးမည်
        """
        return await self.http_client.post(endpoint, json=data)

    async def call_with_retry(self, endpoint: str, data: dict, max_retries: int = 3):
        """
        Retry with exponential backoff
        """
        for attempt in range(max_retries):
            try:
                return await self.http_client.post(endpoint, json=data)
            except Exception as e:
                if attempt == max_retries - 1:
                    raise
                # Exponential backoff: 1s, 2s, 4s
                wait_time = 2 ** attempt
                await asyncio.sleep(wait_time)

    async def call_with_timeout(self, endpoint: str, data: dict, timeout_sec: float = 5.0):
        """
        Timeout: 5 seconds ကျော်ပါက TimeoutError ပေးသည်
        """
        async with asyncio.timeout(timeout_sec):
            return await self.http_client.post(endpoint, json=data)
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **Synchronous** communication သည် immediate response လိုသောနေရာ (user-facing operations) တွင် သင့်တော်သည်၊ သို့သော် temporal coupling (ကာလပိုင်း မှီခိုမှု) ဖြစ်ပေါ်သည်။
2. **Asynchronous** messaging သည် high availability, loose coupling, high throughput လိုသောနေရာ (background processing, notifications) တွင် သင့်တော်သည်။
3. **Consistency-Availability-Coupling** triangle — မည်သည်ကို ဦးစားပေးမည်ဆိုသည်ကို system requirements ပေါ် မူတည်ပြီး communication pattern ရွေးချယ်ရသည်။
4. **Hybrid Architecture** — Real-world systems တွင် sync နှင့် async ကို ပေါင်းစပ်သုံးသည်; user-facing operation sync, background processing async ဆိုသည့် pattern ကို ကျင့်သုံးပါ။
5. **Resilience patterns** — Circuit Breaker, Retry with Backoff, Timeout တို့ကို synchronous calls တိုင်းတွင် ထည့်သွင်းစဉ်းစားပါ။
6. **Protocol ရွေးချယ်မှု** — latency requirement, payload size, type safety, streaming needs, team familiarity တို့ ပေါ် မူတည်ပြီး use case ပေါ်မှ ဆုံးဖြတ်ပါ — "one size fits all" မဟုတ်ပါ။
