# အခန်း ၁၂ — Saga Pattern — Distributed Transactions

## နိဒါန်း

Microservices architecture တွင် business process တစ်ခုသည် မကြာခဏ service အများစုကို ဖြတ်ကျော်ဆောင်ရွက်ရသည်။ ဥပမာ — e-commerce ၏ order placement process တွင် Order Service, Payment Service, Inventory Service နှင့် Notification Service တို့ တပြိုင်နက် ပါဝင်ရသည်။ Monolithic application တွင် database transaction တစ်ခုဖြင့် ACID ကို သေချာပေါ်လွင်ကာ atomicity ကို ရရှိနိုင်သော်လည်း microservices တွင် service တစ်ခုချင်းစီ ၎င်း၏ database ကိုယ်ပိုင်ဆောင်ထားသဖြင့် ဤနည်းလမ်း မသင့်တော်ပါ။ **Saga Pattern** သည် ဤပြဿနာကို **compensating transactions** ကို သုံး၍ ဖြေရှင်းသည်။ ဤအခန်းတွင် Two-Phase Commit ၏ ကန့်သတ်ချက်များ၊ Saga ၏ choreography နှင့် orchestration approach နှစ်မျိုး နှင့် compensating transaction design ကို ဆွေးနွေးမည်။

---

## ၁၂.၁ Two-Phase Commit Fails in Microservices

**Two-Phase Commit (2PC)** သည် distributed transaction ၏ ရိုးရာ approach ဖြစ်သည်။ Coordinator service တစ်ခုက participant services များကို "prepare" ဟု မေးကာ အားလုံး ok ဖြစ်မှ "commit" လုပ်သည်။

```
Phase 1 (Prepare):
Coordinator ──→ Service A: "Prepare?"  → "Ready ✅"
            ──→ Service B: "Prepare?"  → "Ready ✅"
            ──→ Service C: "Prepare?"  → "Ready ✅"

Phase 2 (Commit):
Coordinator ──→ Service A: "Commit!" → Done
            ──→ Service B: "Commit!" → Done
            ──→ Service C: "Commit!" → Done
```

### ဘာကြောင့် Microservices တွင် မသုံးနိုင်သနည်း

```
ပြဿနာ ၁: Network Partition
Coordinator ──→ Service A: commit ✅
            ──→ Service B: [timeout! network down ❌]
            ──→ Service C: commit ✅
Result: A နှင့် C commit ပြီး B မသိ → inconsistent state

ပြဿနာ ၂: Blocking
Prepare phase တွင် resources lock လုပ်ထားသည်
Commit ကြာကြာ မဆုံးလျှင် → blocking → performance ပြဿနာ

ပြဿနာ ၃: Single Point of Failure
Coordinator ကြောင့် failure ဖြစ်လျှင် system ရပ်တန့်သည်

ပြဿနာ ၄: Heterogeneous Systems
NoSQL database, message broker, external API တို့သည်
2PC protocol ကို support မလုပ်ပါ
```

---

## ၁၂.၂ Choreography-Based Sagas

**Choreography** တွင် central coordinator မရှိဘဲ service တစ်ခုချင်းစီသည် event ကို listen ၍ ၎င်း၏ local action ကို လုပ်ဆောင်ပြီး next event ကို publish လုပ်သည်။

### E-Commerce Order Saga (Choreography)

```
Order Placed Event
        │
        ↓
[Payment Service]
  → Deduct payment ✅
  → Publishes: PaymentProcessed Event
        │
        ↓
[Inventory Service]
  → Reserve stock ✅
  → Publishes: StockReserved Event
        │
        ↓
[Shipping Service]
  → Schedule shipment ✅
  → Publishes: ShipmentScheduled Event
        │
        ↓
[Notification Service]
  → Send confirmation email ✅
  → Publishes: OrderConfirmed Event
```

### Failure Scenario with Compensating Transactions

```
Order Placed Event
        │
        ↓
[Payment Service] ✅ → PaymentProcessed
        │
        ↓
[Inventory Service] ❌ (Out of stock!)
  → Publishes: StockReservationFailed Event
        │
        ↓
[Payment Service] (listens to StockReservationFailed)
  → Refund payment ✅   ← compensating transaction
  → Publishes: PaymentRefunded Event
        │
        ↓
[Order Service] (listens to PaymentRefunded)
  → Cancel order
  → Notify customer
```

```python
# Choreography Saga — Payment Service ဥပမာ
class PaymentService:

    @event_handler('OrderPlaced')
    def on_order_placed(self, event):
        """အမှာစာ တင်သောအခါ payment process"""
        try:
            payment = self.payment_gateway.charge(
                customer_id=event.data['customerId'],
                amount=event.data['totalAmount']
            )
            # success event publish
            self.event_bus.publish(PaymentProcessedEvent(
                order_id=event.data['orderId'],
                payment_id=payment.id,
                amount=payment.amount
            ))
        except PaymentFailedError as e:
            # failure event publish — inventory/shipping ကို မသိ
            self.event_bus.publish(PaymentFailedEvent(
                order_id=event.data['orderId'],
                reason=str(e)
            ))

    @event_handler('StockReservationFailed')
    def on_stock_reservation_failed(self, event):
        """Stock မရှိသောကြောင့် payment ပြန်အမ်းသည် — compensating tx"""
        self.payment_gateway.refund(
            payment_id=event.data['paymentId'],
            reason='Out of stock'           # ပြန်အမ်းသောအကြောင်း
        )
        self.event_bus.publish(PaymentRefundedEvent(
            order_id=event.data['orderId']
        ))
```

---

## ၁၂.၃ Orchestration-Based Sagas

**Orchestration** တွင် **Saga Orchestrator** ဟုခေါ်သော central service တစ်ခုက saga ၏ flow ကို ကိုင်တွယ်ကာ participant services ကို command ပေးသည်။

### Saga Orchestrator Pattern

```
                  ┌─────────────────────┐
                  │   Saga Orchestrator │
                  │  (Order Saga)       │
                  └──────────┬──────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ↓                 ↓                 ↓
   [Payment Svc]    [Inventory Svc]    [Shipping Svc]
   (command/reply) (command/reply)   (command/reply)
```

```python
# Order Saga Orchestrator
class OrderSagaOrchestrator:

    def __init__(self):
        self.state = 'STARTED'              # saga state

    def start(self, order_id):
        """Saga ကို စတင်သည်"""
        self.order_id = order_id
        self.state = 'PAYMENT_PENDING'

        # Payment Service ကို command ပို့သည်
        self.command_bus.send(ProcessPaymentCommand(
            order_id=order_id,
            reply_channel='saga.order.reply'  # response channel
        ))

    def on_payment_processed(self, reply):
        """Payment အောင်မြင်သောအခါ"""
        self.payment_id = reply.payment_id
        self.state = 'INVENTORY_PENDING'

        # Inventory Service ကို command ပို့သည်
        self.command_bus.send(ReserveStockCommand(
            order_id=self.order_id,
            items=self.order_items
        ))

    def on_payment_failed(self, reply):
        """Payment မအောင်မြင်သောအခါ"""
        self.state = 'FAILED'
        self.command_bus.send(CancelOrderCommand(
            order_id=self.order_id,
            reason=f"Payment failed: {reply.reason}"
        ))

    def on_stock_reserved(self, reply):
        """Stock ယူထားနိုင်သောအခါ"""
        self.state = 'SHIPPING_PENDING'
        self.command_bus.send(ScheduleShipmentCommand(
            order_id=self.order_id,
            address=self.delivery_address
        ))

    def on_stock_reservation_failed(self, reply):
        """Stock မရှိသောကြောင့် payment ပြန်အမ်း — compensate"""
        self.state = 'COMPENSATING'
        # Payment ပြန်အမ်းရမည်
        self.command_bus.send(RefundPaymentCommand(
            payment_id=self.payment_id,
            reason='Out of stock'
        ))

    def on_payment_refunded(self, reply):
        """Refund ပြီးနောက် saga ပြီးဆုံးသည်"""
        self.state = 'COMPENSATED'
        self.command_bus.send(CancelOrderCommand(
            order_id=self.order_id,
            reason='Compensation complete'
        ))

    def on_shipment_scheduled(self, reply):
        """Shipment ဆောင်ရွက်ပြီး — saga အောင်မြင်"""
        self.state = 'COMPLETED'
        self.command_bus.send(ConfirmOrderCommand(
            order_id=self.order_id
        ))
```

---

## ၁၂.၄ Compensating Transactions

**Compensating Transaction** သည် ပြီးသွားပြီးသော step ၏ effect ကို undo လုပ်သော operation ဖြစ်သည်။ SQL rollback ကဲ့သို့ instantaneous မဟုတ်ဘဲ business-level undo ဖြစ်သည်။

```
Saga Steps ↓                    Compensating Steps ↑
─────────────────────────────────────────────────────
T1: Create Order          ←→  Cancel Order
T2: Reserve Inventory     ←→  Release Inventory
T3: Charge Customer       ←→  Refund Customer
T4: Schedule Shipment     ←→  Cancel Shipment
T5: Send Confirmation     ←→  Send Cancellation Email
```

### Compensating Transactions ၏ သဘောတရား

```python
# Compensation ဥပမာ — Inventory Service
class InventoryService:

    def reserve_stock(self, order_id, items):
        """Forward transaction — stock ယူထားသည်"""
        for item in items:
            self.inventory.decrement(
                product_id=item.product_id,
                quantity=item.qty
            )
        # reservation record သိမ်းသည် (compensation အတွက်)
        self.reservations.save(Reservation(
            order_id=order_id,
            items=items,
            reserved_at=datetime.utcnow()
        ))

    def release_stock(self, order_id):
        """Compensating transaction — stock ပြန်ထည့်သည်"""
        reservation = self.reservations.get(order_id)
        if not reservation:
            return  # idempotent — already released

        for item in reservation.items:
            self.inventory.increment(
                product_id=item.product_id,
                quantity=item.qty         # ဆွဲထားသောပမာဏ ပြန်ထည့်
            )
        self.reservations.mark_released(order_id)
```

---

## ၁၂.၅ Saga State Machine Design

Saga ကို **state machine** ပုံစံဖြင့် ဒီဇိုင်းချခြင်းသည် saga ၏ current state ကို track လုပ်ရန် အကောင်းဆုံးဖြစ်သည်။

```
Order Saga State Machine:
─────────────────────────────────────────────────────────
         ┌──────────┐
    ────→│ STARTED  │
         └────┬─────┘
              │ payment cmd
              ↓
    ┌─────────────────┐    payment failed    ┌──────────┐
    │ PAYMENT_PENDING │──────────────────────→│  FAILED  │
    └────────┬────────┘                       └──────────┘
             │ payment ok
             ↓
    ┌──────────────────┐   stock failed    ┌──────────────────┐
    │ INVENTORY_PENDING│──────────────────→│  COMPENSATING    │
    └────────┬─────────┘                   │ (refund pending) │
             │ stock ok                    └────────┬─────────┘
             ↓                                      │ refund done
    ┌──────────────────┐                            ↓
    │ SHIPPING_PENDING │                    ┌──────────────┐
    └────────┬─────────┘                    │ COMPENSATED  │
             │ shipping ok                  └──────────────┘
             ↓
         ┌───────────┐
         │ COMPLETED │
         └───────────┘
```

```python
# Saga state machine with persistence
from enum import Enum

class OrderSagaState(Enum):
    STARTED = "STARTED"
    PAYMENT_PENDING = "PAYMENT_PENDING"
    INVENTORY_PENDING = "INVENTORY_PENDING"
    SHIPPING_PENDING = "SHIPPING_PENDING"
    COMPLETED = "COMPLETED"
    COMPENSATING = "COMPENSATING"
    COMPENSATED = "COMPENSATED"
    FAILED = "FAILED"

class OrderSagaInstance:
    """Database ထဲ persist ဖြစ်သော saga instance"""

    def __init__(self, saga_id, order_id):
        self.saga_id = saga_id
        self.order_id = order_id
        self.state = OrderSagaState.STARTED
        self.payment_id = None
        self.created_at = datetime.utcnow()
        self.updated_at = datetime.utcnow()

    def transition_to(self, new_state):
        """State transition ကို log တင်ကာ update လုပ်သည်"""
        print(f"Saga {self.saga_id}: {self.state} → {new_state}")
        self.state = new_state
        self.updated_at = datetime.utcnow()
        self._save()             # DB persist
```

---

## ၁၂.၆ Choreography vs Orchestration — ဘယ်ကို ဘယ်အခါ သုံးမည်

```
┌──────────────────────┬───────────────────────┬────────────────────────┐
│ Aspect               │ Choreography          │ Orchestration          │
├──────────────────────┼───────────────────────┼────────────────────────┤
│ Central Control      │ မရှိ (decentralized)  │ ရှိ (orchestrator)     │
│ Coupling             │ Loose                 │ Orchestrator → services│
│ Visibility           │ ပျောက်ဆုံး (tracking  │ Central state tracking │
│                      │ ခက်)                  │ ဖြစ်နိုင်              │
│ Debugging            │ ခက်ခဲ                 │ လွယ်ကူ                 │
│ Testing              │ ခက်ခဲ                 │ လွယ်ကူ                 │
│ Simple flows         │ ✅ ကောင်း             │ ⚠ overkill            │
│ Complex flows        │ ⚠ ရှုပ်ထွေး          │ ✅ ကောင်း              │
│ Long-running         │ ⚠ ခက်                │ ✅ ကောင်း              │
└──────────────────────┴───────────────────────┴────────────────────────┘
```

### ရွေးချယ်မှုလမ်းညွှန်

```python
# Choreography ကိုရွေးရမည့်အချိန်:
if (saga_steps <= 3 and
    flow_is_linear and
    not need_centralized_monitoring and
    services_owned_by_same_team):
    use_choreography()

# Orchestration ကိုရွေးရမည့်အချိန်:
if (saga_steps > 4 or
    complex_branching_logic or
    need_audit_trail or
    multiple_teams_involved or
    long_running_process):
    use_orchestration()
```

---

## အဓိကသင်ခန်းစာများ

- **Two-Phase Commit** သည် microservices တွင် network partition, blocking, single point of failure ကြောင့် မသင့်တော်ပါ။
- **Saga Pattern** သည် local transactions ကို sequence ပြီး failure အခါ **compensating transactions** ဖြင့် undo ဆောင်ရွက်သည်။
- **Choreography Saga** တွင် services များသည် event ကို listen ပြီး next step ကို publish လုပ်သောကြောင့် coupling နည်းသည်။ သို့သော် complex flow ကို track ရန် ခက်ခဲသည်။
- **Orchestration Saga** တွင် central orchestrator ကပ် saga flow ကို ထိန်းချုပ်သောကြောင့် visibility ကောင်းသော်လည်း orchestrator ပေါ်တွင် coupling ဖြစ်သည်။
- **Compensating Transaction** သည် instant rollback မဟုတ်ဘဲ business-level undo ဖြစ်သောကြောင့် "undo" logic ကို saga design အဆင့်တွင် ဦးစွာ ဆန်းစစ်ရမည်။
- Saga state ကို **State Machine** ပုံစံဖြင့် database တွင် persist လုပ်ကာ failure recovery နှင့် idempotency ကို ဖြေရှင်းပါ။
