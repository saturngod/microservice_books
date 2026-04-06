# နောက်ဆက်တွဲ F: Saga Pattern Templates

---

## F.1 Saga Pattern ဆိုသည်မှာ

Microservices architecture တွင် distributed transaction ကို manage လုပ်ရန် Saga Pattern ကို သုံးသည်။ Single ACID transaction အစား — transaction ကို local transactions တစ်စီးဆီ ခွဲ၍ failure ဖြစ်ပါက compensating transaction ဖြင့် undo လုပ်သည်။

### Saga ၏ နှစ်မျိုး

| မျိုး | အင်္ဂလိပ် | ထိန်းချုပ်သူ | ဆက်သွယ်မှု |
|------|---------|------------|-----------|
| **Choreography** (ကျက်သရေ-ဆိုင်ရာ) | Event-driven, decentralized | Services တစ်ခုချင်းစီ | Events ဖြင့် |
| **Orchestration** (ညှိနှိုင်းချုပ်ကိုင်) | Centralized coordinator | Saga orchestrator | Commands ဖြင့် |

---

## F.2 Choreography Saga — Order Saga Event Flow

### ဥပမာ Scenario: E-commerce Order (Order → Payment → Inventory → Shipping)

#### Happy Path (အောင်မြင်သောလမ်းကြောင်း)

```
Customer        Order Service    Payment Service   Inventory Service  Shipping Service
    │                │                 │                  │                  │
    │─ PlaceOrder ──→│                 │                  │                  │
    │                │─ OrderCreated ─→│                  │                  │
    │                │                 │                  │                  │
    │                │                 │─ PaymentProcessed→│                 │
    │                │                 │                  │                  │
    │                │                 │                  │─ InventoryReserved→│
    │                │                 │                  │                  │
    │                │                 │                  │       ← ShipmentCreated
    │                │                 │                  │                  │
    │          ← OrderCompleted        │                  │                  │
    │ ← 200 OK       │                 │                  │                  │
```

#### Failure Path — Payment ကျ (Compensating Transactions)

```
Customer        Order Service    Payment Service   Inventory Service  Shipping Service
    │                │                 │                  │                  │
    │─ PlaceOrder ──→│                 │                  │                  │
    │                │─ OrderCreated ─→│                  │                  │
    │                │                 │                  │                  │
    │                │              (Payment Failed)       │                  │
    │                │                 │─ PaymentFailed ──→│                 │
    │                │                 │  (compensate)     │                  │
    │                │← PaymentFailed  │                  │                  │
    │                │ (cancel order)  │                  │                  │
    │         ← OrderCancelled         │                  │                  │
    │ ← 4xx/5xx      │                 │                  │                  │
```

#### Choreography Event Flow ဇယား

| Event | Publisher | Subscriber | Action |
|-------|-----------|-----------|--------|
| `OrderCreated` | Order Service | Payment Service | Charge customer |
| `PaymentProcessed` | Payment Service | Inventory Service | Reserve items |
| `PaymentFailed` | Payment Service | Order Service | Cancel order |
| `InventoryReserved` | Inventory Service | Shipping Service | Create shipment |
| `InventoryInsufficient` | Inventory Service | Payment Service | Refund payment |
| `InventoryInsufficient` | Inventory Service | Order Service | Cancel order |
| `ShipmentCreated` | Shipping Service | Order Service | Mark order completed |
| `ShipmentFailed` | Shipping Service | Inventory Service | Release inventory |
| `ShipmentFailed` | Shipping Service | Payment Service | Refund payment |

---

## F.3 Orchestration Saga — Order Saga State Machine

### Orchestrator ၏ State Diagram

```
                    ┌─────────────────────────────────────────────────────┐
                    │              ORDER SAGA ORCHESTRATOR                │
                    └─────────────────────────────────────────────────────┘

     Start
       │
       ▼
  ┌─────────────┐
  │  PENDING    │──── CreateOrder ────→ Order Service
  └─────────────┘
       │ OrderCreated
       ▼
  ┌─────────────┐
  │  PAYING     │──── ProcessPayment ──→ Payment Service
  └─────────────┘
       │                 │ PaymentFailed
       │ PaymentOK       ▼
       │         ┌───────────────────┐
       │         │  CANCELLING_ORDER │──── CancelOrder ──→ Order Service
       │         └───────────────────┘
       │                 │ OrderCancelled
       │                 ▼
       │         ┌───────────────────┐
       │         │    CANCELLED      │ (Terminal State)
       │         └───────────────────┘
       ▼
  ┌─────────────┐
  │  RESERVING  │──── ReserveInventory ──→ Inventory Service
  └─────────────┘
       │                  │ InventoryInsufficient
       │ Inventory OK     ▼
       │         ┌─────────────────────┐
       │         │  REFUNDING_PAYMENT  │──── RefundPayment ──→ Payment Service
       │         └─────────────────────┘
       │                  │ PaymentRefunded
       │                  ▼
       │         ┌───────────────────┐
       │         │  CANCELLING_ORDER │──── CancelOrder ──→ Order Service
       │         └───────────────────┘
       │                  │ OrderCancelled
       │                  ▼
       │         ┌───────────────────┐
       │         │    CANCELLED      │ (Terminal State)
       │         └───────────────────┘
       ▼
  ┌─────────────┐
  │  SHIPPING   │──── CreateShipment ──→ Shipping Service
  └─────────────┘
       │                  │ ShipmentFailed
       │ ShipmentCreated  ▼
       │         ┌────────────────────────┐
       │         │  RELEASING_INVENTORY   │──── ReleaseInventory ──→ Inventory Service
       │         └────────────────────────┘
       │                  │
       │                  ▼
       │         ┌─────────────────────┐
       │         │  REFUNDING_PAYMENT  │
       │         └─────────────────────┘
       │                  │
       │                  ▼
       │         ┌───────────────────┐
       │         │    CANCELLED      │
       │         └───────────────────┘
       ▼
  ┌─────────────┐
  │  COMPLETED  │  (Terminal State — Success)
  └─────────────┘
```

### Orchestration Command / Response ဇယား

| State | Orchestrator sends (Command) | Expected response | Next State (Success) | Next State (Failure) |
|-------|----------------------------|-----------------|---------------------|---------------------|
| PENDING | `CreateOrder` → Order Svc | `OrderCreated` | PAYING | — |
| PAYING | `ProcessPayment` → Payment Svc | `PaymentProcessed` | RESERVING | CANCELLING_ORDER |
| RESERVING | `ReserveInventory` → Inventory Svc | `InventoryReserved` | SHIPPING | REFUNDING_PAYMENT |
| SHIPPING | `CreateShipment` → Shipping Svc | `ShipmentCreated` | COMPLETED | RELEASING_INVENTORY |
| CANCELLING_ORDER | `CancelOrder` → Order Svc | `OrderCancelled` | CANCELLED | retry |
| REFUNDING_PAYMENT | `RefundPayment` → Payment Svc | `PaymentRefunded` | CANCELLING_ORDER | retry |
| RELEASING_INVENTORY | `ReleaseInventory` → Inventory Svc | `InventoryReleased` | REFUNDING_PAYMENT | retry |

---

## F.4 Compensating Transaction ဥပမာ — Banking Transfer Saga

### Scenario: Bank A မှ Bank B သို့ ငွေလွှဲခြင်း

```
Transfer Saga:
  Step 1: Debit $100 from Account A
  Step 2: Credit $100 to Account B
  Step 3: Record transaction log

Compensating Transactions:
  Comp(Step 3): Delete transaction log
  Comp(Step 2): Debit $100 from Account B  (undo credit)
  Comp(Step 1): Credit $100 to Account A   (undo debit)
```

#### Choreography Flow — Banking Transfer

```
                  TransferService
                       │
                       │─── DebitInitiated ──────────────────────────→
                       │                                AccountService-A
                       │                                     │ (Debit $100)
                       │                                     │── DebitCompleted ──→
                       │                                               AccountService-B
                       │                                                    │ (Credit $100)
                       │                                                    │── CreditCompleted ──→
                       │                                               TransactionLogService
                       │                                                    │ (Log record)
                       │                                                    │── TransactionLogged ──→
                       │← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│
                  TransferComplete
```

#### Failure at Credit Step — Compensation Flow

```
                  TransferService
                       │
                       │─── DebitInitiated ──────────────────────────→
                       │                                AccountService-A
                       │                                     │ (Debit $100 ✓)
                       │                                     │── DebitCompleted ──→
                       │                                               AccountService-B
                       │                                                    │ (Credit FAILED ✗)
                       │                                                    │── CreditFailed ──→
                       │                                AccountService-A
                       │                                     │ (Compensate: Credit $100 back)
                       │                                     │── DebitCompensated ──→
                       │← TransferFailed ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│
```

#### Banking Transfer Compensating Transaction ဇယား

| Transaction | Compensating Transaction | Idempotency Key | မှတ်ချက် |
|------------|------------------------|-----------------|---------|
| `DebitAccount(A, $100)` | `CreditAccount(A, $100)` | `txn_id + "comp"` | Exact reversal |
| `CreditAccount(B, $100)` | `DebitAccount(B, $100)` | `txn_id + "comp"` | Exact reversal |
| `RecordTransaction(log)` | `DeleteTransaction(log)` | `txn_id` | Idempotent delete |
| `NotifyCustomer(email)` | — | — | Notification undo မဖြစ်နိုင် (side effect) |

> **သတိ:** Notification ကဲ့သို့သော side effects (email ပို့ပြီး) ကို undo မလုပ်နိုင်ပါ။ ထို့ကြောင့် saga design တွင် side effects ကို နောက်ဆုံးတွင် ထည့်ပါ။

---

## F.5 Choreography vs Orchestration ရွေးချယ်မှု ဆုံးဖြတ်ဇယား

| ဆုံးဖြတ်မှုသတ်မှတ်ချက် | **Choreography** | **Orchestration** |
|----------------------|----------------|-----------------|
| **Coupling** | Services တစ်ခုနှင့်တစ်ခု directly မဆက် (loosely coupled) | Orchestrator နှင့် services တွဲနေသည် |
| **Visibility** | ခက်ခဲသည် — distributed tracing လိုအပ် | Orchestrator တွင် state အပြည့်မြင်ရသည် |
| **Complexity** | Saga logic ကို services တွင် ဖြန့်ကြဲ | Orchestrator တစ်ခုတည်းတွင် centralized |
| **Debugging** | ခက်ခဲ — event chain ကို follow လုပ်ရ | ပိုလွယ်ကူ — orchestrator state ကို ကြည့်ရ |
| **Testing** | Unit test ပိုမိုလွတ်လပ်; integration test ခက် | Orchestrator unit test လွယ်; end-to-end test ပိုများ |
| **Service failure handling** | Service တစ်ခုချင်း retry logic တည်ဆောက် | Orchestrator ဗဟိုတွင် retry policy သတ်မှတ် |
| **Versioning** | Event schema versioning ခက်ခဲ | Orchestrator version bump ဖြင့် ပိုကောင်း |
| **Scalability** | Service တစ်ခုချင်းစီ scale သီးသန့်ဖြစ်နိုင်သည် | Orchestrator ကိုယ်တိုင် bottleneck ဖြစ်နိုင် |
| **Technology** | Event broker (Kafka, RabbitMQ) | Workflow engine (Temporal, Conductor, Camunda) |
| **Best for** | Simple, well-defined, stable workflows | Complex, long-running, stateful processes |

### ရွေးချယ်မှု Guideline

```
Choreography ကို ရွေးပါ:
  ✓ Services 3-4 ခုသာ ပါဝင်သည်
  ✓ Event-driven architecture ရှိပြီးသား
  ✓ Services independent deployment ဦးစားပေးသည်
  ✓ Simple linear flow (branching မပါ)
  ✓ Team တစ်ခုချင်းစီ service ကိုယ်ပိုင်ဆောင်ရွက်မည်

Orchestration ကို ရွေးပါ:
  ✓ Complex branching logic ရှိသည်
  ✓ Long-running processes (hours/days)
  ✓ State visibility / audit log အရေးကြီးသည်
  ✓ Retry, timeout, compensation logic ရှုပ်ထွေးသည်
  ✓ Services 5+ ခုနှင့် parallel execution ပါဝင်သည်
  ✓ Business process monitoring လိုအပ်သည်
```

---

## F.6 Saga Implementation Tools

| Tool / Framework | Saga Type | Language | မှတ်ချက် |
|-----------------|-----------|---------|---------|
| **Temporal** | Orchestration | Go, Java, Python, .NET, TypeScript | Workflow engine; durable execution |
| **Netflix Conductor** | Orchestration | Java (server); any client | REST-based workflow definition |
| **Camunda** | Orchestration | Java | BPMN-based; enterprise features |
| **Axon Framework** | Choreography + Orchestration | Java | DDD-focused; event sourcing built-in |
| **Eventuate Tram** | Choreography | Java | Microservices patterns library |
| **MassTransit** | Choreography + Orchestration | .NET | State machine sagas (Automatonymous) |
| **Spring State Machine** | Orchestration | Java/Spring | State machine library |
| **Custom (Kafka + DB)** | Choreography | Any | Outbox pattern + event sourcing |

---

## F.7 Saga ကောင်းမွန်မှုအတွက် Design Checklist

- [ ] Transaction တစ်ခုချင်း **idempotent** ဖြစ်သည် (retry လုပ်သောအခါ side effect မဖြစ်)
- [ ] Compensating transaction တစ်ခုချင်း သတ်မှတ်ထားသည်
- [ ] Saga state ကို durable storage တွင် သိမ်းသည် (saga log)
- [ ] **Outbox pattern** ဖြင့် event publishing ရင်ပြင်သည် (dual write ကာကွယ်)
- [ ] Dead letter handling — compensation ကိုယ်တိုင် fail ဖြစ်လျှင် alert + manual intervention
- [ ] Idempotency key (correlation ID) တစ်ခုချင်းစီ saga step တွင် pass ဖြင့် ပါ
- [ ] Circuit breaker ဖြင့် cascading failure ကာကွယ်ထားသည်
- [ ] Saga timeout — အချိန်ကြာလွန်းလျှင် auto-cancel logic ရှိသည်
- [ ] Distributed tracing (trace ID) saga steps တစ်ခုချင်းစီ propagate ဖြင့် ပါ
