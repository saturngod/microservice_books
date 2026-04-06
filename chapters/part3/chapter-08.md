# အခန်း ၈ — Event-Driven Architecture အခြေခံများ

## နိဒါန်း

Software architecture လောကတွင် system များကြား ဆက်သွယ်မှုပုံစံသည် ကြီးမားသော အကျိုးသက်ရောက်မှုများ ရှိသည်။ ရိုးရာ request-response ပုံစံမှ ဖြစ်ပေါ်လာသော **Event-Driven Architecture (EDA)** သည် microservices ခေတ်တွင် အလွန်အရေးပါသော pattern တစ်ခုဖြစ်သည်။ EDA တွင် service များသည် တစ်ခုနှင့်တစ်ခု တိုက်ရိုက် ခေါ်ဆိုမည့်အစား event များ ထုတ်ပြန်ပြီး၊ သက်ဆိုင်ရာ service များက ၎င်း event များကို subscribe လုပ်ကာ တုံ့ပြန်သည်။ ဤနည်းလမ်းသည် service များကြား loose coupling ဖြစ်စေပြီး scalability နှင့် resilience ကို သိသာစွာ တိုးတက်စေသည်။ ဤအခန်းတွင် EDA ၏ အခြေခံသဘောတရားများ၊ event အမျိုးအစားများ၊ design နည်းများနှင့် trade-off များကို အသေးစိတ် လေ့လာမည်။

---

## ၈.၁ Events vs Commands vs Queries

Microservices ကို design လုပ်သောအခါ message သုံးမျိုးကို ရှင်းရှင်းလင်းလင်း ကွဲပြားနားလည်ရမည်။

### Event

**Event** ဆိုသည်မှာ ဖြစ်သွားပြီးသော အဖြစ်အပျက်တစ်ခုကို ကြေညာချက် (notification) ပုံစံဖြင့် ဖော်ပြသည့် message ဖြစ်သည်။ Event သည် "ဘာဖြစ်သွားသည်" ကို ဖော်ပြပြီး မည်သူ respond လုပ်ရမည်ကို သတ်မှတ်မထားပါ။

```json
// OrderPlaced event - အမှာစာ တင်သွင်းပြီးကြောင်း event
{
  "eventType": "OrderPlaced",       // event အမျိုးအစား
  "eventId": "evt-12345",           // unique identifier
  "occurredAt": "2026-04-06T10:00:00Z", // ဖြစ်ပေါ်သည့်အချိန်
  "data": {
    "orderId": "ord-9876",
    "customerId": "cust-001",
    "totalAmount": 59900
  }
}
```

### Command

**Command** သည် တစ်ခုခုကို လုပ်ဆောင်ပေးရန် တောင်းဆိုချက်ဖြစ်သည်။ Command တစ်ခုကို receiver တစ်ဦးတည်းသို့ ညွှန်ကြားလျှောက်ထားသည်။ Command ကို reject လုပ်နိုင်သည်။

```json
// PlaceOrder command - အမှာစာ တင်သွင်းပေးရန် ညွှန်ကြားချက်
{
  "commandType": "PlaceOrder",
  "commandId": "cmd-99887",
  "targetService": "OrderService",
  "payload": {
    "customerId": "cust-001",
    "items": [{"productId": "prod-1", "qty": 2}]
  }
}
```

### Query

**Query** သည် data တောင်းယူမှုဖြစ်ပြီး side effect မဖြစ်ပေါ်စေပါ။ response ကို မျှော်မှန်းသည်။

```
// Query မျိုး - read-only ဖြစ်သည်
GET /orders/{orderId}   // state ကို မပြောင်းလဲ
```

### ကွာခြားချက် နှိုင်းယှဉ်ချက်

```
┌─────────────┬──────────────────┬─────────────────┬────────────────┐
│             │ Event            │ Command         │ Query          │
├─────────────┼──────────────────┼─────────────────┼────────────────┤
│ တိသို့ မှ  │ 1 → Many         │ 1 → 1           │ 1 → 1          │
│ ဖြစ်ပုံ    │ ဖြစ်ပြီးသား     │ ဖြစ်ပေးရန်တောင်း│ ဖတ်ရှုရန်      │
│ Reject?    │ မဖြစ်နိုင်       │ ဖြစ်နိုင်       │ မဖြစ်နိုင်     │
│ Side Effect│ မမျှော်မှန်း    │ မျှော်မှန်း    │ မရှိ           │
└─────────────┴──────────────────┴─────────────────┴────────────────┘
```

---

## ၈.၂ Event Notification vs Event-Carried State Transfer

Event pattern နှစ်မျိုးကို ရှင်းလင်းစွာ ကွဲပြားနားလည်ရမည်။

### Event Notification (အကြောင်းကြားချက် Event)

Event ၏ payload တွင် ဖြစ်ပေါ်မှုကြောင်း ကြေညာချက်သာ ပါဝင်ပြီး၊ လိုအပ်သော data ကို consumer ကိုယ်တိုင် ထပ်မံ ဆောင်ရွက်ရသည်။

```json
// Event Notification ပုံစံ - data နည်းသည်
{
  "eventType": "CustomerEmailChanged",
  "customerId": "cust-001"
  // email အသစ်ပါမထည့် — consumer ကိုယ်တိုင် ဆွဲယူရမည်
}
```

**ဆုတ်ကျိုးချက်:** Consumer သည် CustomerService ကို API call ထပ်မံ ချိတ်ဆက်ရမည်ဖြစ်သဖြင့် temporal coupling ဖြစ်နိုင်သည်။

### Event-Carried State Transfer (Data ပါဆောင်သော Event)

Event payload ထဲတွင် လိုအပ်သော data အားလုံး ပါဝင်သည်။ Consumer သည် ထပ်မံ API call မခေါ်ဘဲ တုံ့ပြန်နိုင်သည်။

```json
// Event-Carried State Transfer ပုံစံ - data ပြည့်ပြည့်ဝဝ ပါသည်
{
  "eventType": "CustomerEmailChanged",
  "customerId": "cust-001",
  "previousEmail": "old@example.com",  // မူလ email
  "newEmail": "new@example.com",       // အသစ် email
  "changedAt": "2026-04-06T10:00:00Z"
}
```

**ကောင်းသောချက်:** Consumer service များသည် CustomerService နှင့် decoupled ဖြစ်သည်။ **ဆုတ်ကျိုးချက်:** Event payload ကြီးမားလာနိုင်ပြီး ဒေတာ ထပ်တလဲလဲ ဖြစ်နိုင်သည်။

```
Event Notification              Event-Carried State Transfer
────────────────────            ─────────────────────────────
Producer → Event (ID only)      Producer → Event (Full Data)
Consumer → API Call (fetch)     Consumer → Process directly
         → CustomerService                 (no extra call)
```

---

## ၈.၃ Domain Events vs Integration Events vs System Events

### Domain Events

**Domain Events** သည် business domain အတွင်းမှ ဖြစ်ပေါ်သော အရေးကြီးသော ဖြစ်ရပ်များဖြစ်သည်။ ၎င်းတို့သည် business language ဖြင့် ဖော်ပြသည်။

```csharp
// Domain Event ဥပမာ (C#)
public class OrderConfirmedEvent
{
    public string OrderId { get; init; }         // အမှာစာ ID
    public string CustomerId { get; init; }      // ဖောက်သည် ID
    public decimal TotalAmount { get; init; }    // စုစုပေါင်းငွေပမာဏ
    public DateTime ConfirmedAt { get; init; }   // အတည်ပြုသည့်အချိန်
    public List<OrderItem> Items { get; init; }  // ပစ္စည်းစာရင်း
}
```

### Integration Events

**Integration Events** သည် bounded context နှစ်ခု သို့မဟုတ် service နှစ်ခုကြား ဆက်သွယ်ရန် အသုံးပြုသော event များဖြစ်သည်။ ၎င်းတို့သည် မကြာခဏ domain event မှ transform ဖြစ်သည်။

```python
# Integration Event - service အများကြား ပြောဆိုသော event
class OrderShippedIntegrationEvent:
    def __init__(self):
        self.event_type = "order.shipped"           # event type
        self.order_id = None                        # အမှာစာ ID
        self.tracking_number = None                 # tracking နံပါတ်
        self.estimated_delivery = None              # မျှော်မှန်းချိန်
        self.schema_version = "1.0"                 # version (backward compat)
```

### System Events

**System Events** သည် infrastructure သို့မဟုတ် technical layer မှ ဖြစ်ပေါ်သော event များဖြစ်သည်။

```json
{
  "eventType": "ServiceHealthDegraded",    // system event
  "serviceId": "payment-service",
  "severity": "WARNING",
  "metric": "error_rate",
  "currentValue": 0.15,
  "threshold": 0.05
}
```

---

## ၈.၄ Event Schema Design & Backward Compatibility

Event schema ကို ထိန်းသိမ်းသည့်အခါ backward compatibility ဖြစ်ရေးသည် အလွန်အရေးကြီးသည်။ Consumer များ မပြောင်းဘဲ producer ကို update လုပ်နိုင်ရမည်။

### Versioning Strategies

```json
// v1 schema
{
  "schemaVersion": "1.0",
  "eventType": "ProductCreated",
  "productId": "prod-001",
  "name": "Widget",
  "price": 9900
}

// v2 schema — field အသစ် optional ထည့်သည် (backward compatible)
{
  "schemaVersion": "2.0",
  "eventType": "ProductCreated",
  "productId": "prod-001",
  "name": "Widget",
  "price": 9900,
  "currency": "MMK",       // field အသစ် — optional
  "category": "electronics" // field အသစ် — optional
}
```

### ကမ်းလှမ်းချက်များ (Schema Evolution Rules)

```
✅ Safe Changes (Backward Compatible):
   - Optional field အသစ် ထည့်ခြင်း
   - Field ၏ description ပြောင်းခြင်း
   - Enum တွင် တန်ဖိုးအသစ် ထည့်ခြင်း

❌ Breaking Changes (Avoid!):
   - Field ဖျက်ခြင်း
   - Field အမည် ပြောင်းခြင်း
   - Field ၏ data type ပြောင်းခြင်း
   - Required field ထည့်ခြင်း
```

---

## ၈.၅ Idempotency & At-Least-Once Delivery

### At-Least-Once Delivery

Distributed system တွင် message delivery guarantee ကို သေချာပေါက် once-only အဖြစ် ရရှိရန် ခက်ခဲသည်။ Most broker များသည် **at-least-once** delivery ကို ပေးသည်၊ ဆိုလိုသည်မှာ တူညီသော event ကို တစ်ကြိမ်ထက်ပိုပြီး receive ဖြစ်နိုင်သည်။

### Idempotency (တူညီသော operation ကို ထပ်ကြိမ်လုပ်လျှင် ရလဒ်မပြောင်းလဲမှု)

```python
# Idempotent consumer ဥပမာ
class OrderProcessor:
    def process_order_placed(self, event):
        event_id = event["eventId"]

        # တူညီ event ကို ထပ်ကြိမ် process မဖြစ်ရအောင် စစ်ဆေးသည်
        if self.processed_events.contains(event_id):
            print(f"Event {event_id} already processed, skipping")
            return  # duplicate ဖြစ်သဖြင့် skip

        # business logic လုပ်ဆောင်
        self.create_order(event["data"])

        # processed ဖြစ်ကြောင်း မှတ်တမ်းတင်
        self.processed_events.add(event_id, ttl=86400)  # 24 နာရီ သိမ်းဆည်း
```

```
Event ၁ (ID: abc) ──→ Consumer ──→ Process ──→ Mark as processed
                                                    │
Event ၁ (ID: abc) ──→ Consumer ──→ Check ID ────→ Skip (duplicate)
  (ထပ်ရောက်လာ)                       found!
```

---

## ၈.၆ Event-Driven vs Request-Driven Trade-offs

### Request-Driven Architecture

```
Client ──→ Service A ──→ Service B ──→ Service C
                                           │
                                     (ပြန်ဖြေ)
                ←─────────────────────────┘
```

**ကောင်းသောချက်:** Simple, debuggable, immediate response
**ဆုတ်ကျိုးချက်:** Tight coupling, cascading failures

### Event-Driven Architecture

```
Service A ──→ [Event Bus] ──→ Service B
                         └──→ Service C
                         └──→ Service D
   (ကြေညာ)    (distribute)   (respond independently)
```

**ကောင်းသောချက်:** Loose coupling, scalability, resilience
**ဆုတ်ကျိုးချက်:** Complex debugging, eventual consistency

### နှိုင်းယှဉ်ချက်

```
┌────────────────────┬──────────────────┬─────────────────────┐
│ Aspect             │ Request-Driven   │ Event-Driven        │
├────────────────────┼──────────────────┼─────────────────────┤
│ Coupling           │ Tight            │ Loose               │
│ Debugging          │ လွယ်ကူ           │ ခက်ခဲ               │
│ Scalability        │ ကန့်သတ်           │ ကောင်းမွန်           │
│ Consistency        │ Strong           │ Eventual            │
│ Latency            │ Synchronous      │ Asynchronous        │
│ Failure handling   │ Cascade risk     │ Isolated            │
└────────────────────┴──────────────────┴─────────────────────┘
```

---

## အဓိကသင်ခန်းစာများ

- **Event** သည် ဖြစ်ပြီးသော fact ဖြစ်ပြီး immutable ဖြစ်သည်။ **Command** သည် တောင်းဆိုချက်ဖြစ်ပြီး reject ဖြစ်နိုင်သည်။
- **Event Notification** သည် payload သေးငယ်သော်လည်း consumer ကို extra API call ဖြစ်စေသည်။ **Event-Carried State Transfer** သည် ကိုယ်ပိုင် data ပြည့်ပြည့်ဝဝ ဆောင်သဖြင့် decoupled ဖြစ်သည်။
- **Domain Events** သည် business language ဖြင့် ဖော်ပြပြီး bounded context အတွင်းတွင် ဖြစ်သည်။ **Integration Events** သည် service boundary ကို ဖြတ်ကျော်သည်။
- Schema evolution တွင် optional field ထည့်ခြင်းသည် safe ဖြစ်သော်လည်း field ဖျက်ခြင်း သို့မဟုတ် rename ခြင်းသည် breaking change ဖြစ်သည်။
- At-least-once delivery ကြောင့် consumer များသည် **idempotent** ဖြစ်ရမည်။ event ID ကို စစ်ဆေး၍ duplicate ကို ကာကွယ်ရမည်။
- Event-Driven Architecture သည် scalability နှင့် resilience ကောင်းသော်လည်း eventual consistency နှင့် debugging complexity ကို ဂရုစိုက်ရသည်။
