# အခန်း ၂ — Microservices အတွက် Domain-Driven Design

## နိဒါန်း

Domain-Driven Design (DDD) သည် Eric Evans ၂၀၀၃ ခုနှစ်တွင် ရေးသားသော "Domain-Driven Design: Tackling Complexity in the Heart of Software" စာအုပ်မှ ဆင်းသက်လာသော software design philosophy တစ်ရပ်ဖြစ်သည်။ Microservices architecture တွင် service boundaries ကို မှန်ကန်စွာ သတ်မှတ်နိုင်ရန် DDD သည် မရှိမဖြစ် လမ်းညွှန်ချက်တစ်ခုဖြစ်သည်။ DDD မပါဘဲ Microservices ဆောက်ပါက services တွင် chatty communication (အပြန်အလှန် မကြာမကြာ ဆက်သွယ်မှု)၊ circular dependencies (ပတ်လည် မှီခိုမှု)၊ နှင့် inappropriate service granularity (service ကြီးငယ်မမျှမတ) တို့ ဖြစ်ပေါ်တတ်သည်။ ဤအခန်းတွင် DDD ၏ သဘောတရားများ၊ Microservices နှင့် ဘယ်လို ပေါင်းစပ်သုံးစွဲမည်ဆိုသည်ကို လေ့လာသွားမည်ဖြစ်သည်။

---

## ၂.၁ Ubiquitous Language နှင့် Bounded Contexts

### Ubiquitous Language

**Ubiquitous Language** (ကမ္ဘာ့ဘာသာ / ဘုံသုံးဘာသာ) ဆိုသည်မှာ developer များနှင့် domain expert (business stakeholder) တို့ပူးတွဲသုံးသော တူညီသောစကားလုံးများနှင့် ဘာသာစကားဖြစ်သည်။

ပြဿနာ — တူညီသောအရာကို ကွဲပြားသောနာမ်ဖြင့် ခေါ်ဆိုမှု

```
Business Team က သုံးသော နာမ် | Developer က သုံးသော နာမ်
─────────────────────────────────────────────────────────
"Customer"                    | "User" / "Client" / "Account"
"Order"                       | "Transaction" / "Purchase"
"Cart"                        | "BasketSession" / "TempOrder"
```

Ubiquitous Language ကျင့်သုံးပါက ―

```python
# ❌ မှားသော နာမ်ပေး (developer language ကိုသာ ကြည့်ရှု)
class UserTransactionBasket:
    def process_purchase_flow(self):
        pass

# ✅ မှန်ကန်သော နာမ်ပေး (domain language ကို ရောင်ပြန်ဟပ်)
class CustomerOrder:
    def place_order(self):
        pass
```

### Bounded Context

**Bounded Context** ဆိုသည်မှာ domain model တစ်ခု valid ဖြစ်သည့် boundary (နယ်ပယ်) တစ်ခုဖြစ်သည်။ "Product" ဟူသောစကားလုံးသည် context ကွဲပါက သဘောတရားကွဲနိုင်သည်။

```
┌─────────────────────────────────────────────────────────────┐
│                      E-Commerce Domain                       │
│                                                             │
│  ┌─────────────────┐         ┌─────────────────────────┐   │
│  │ Catalog Context │         │    Order Context        │   │
│  │                 │         │                         │   │
│  │  Product:       │         │  Product:               │   │
│  │  - name         │         │  - product_id           │   │
│  │  - description  │         │  - quantity_ordered     │   │
│  │  - images       │         │  - unit_price_at_order  │   │
│  │  - category     │         │  (snapshot ကို သိမ်းသည်)  │   │
│  └─────────────────┘         └─────────────────────────┘   │
│                                                             │
│  ┌─────────────────┐         ┌─────────────────────────┐   │
│  │ Shipping Context│         │   Payment Context       │   │
│  │                 │         │                         │   │
│  │  Product:       │         │  (Product မသိပါ)         │   │
│  │  - weight       │         │  Transaction            │   │
│  │  - dimensions   │         │  - amount               │   │
│  │  - hazardous?   │         │  - currency             │   │
│  └─────────────────┘         └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

"Product" ၏ အဓိပ္ပာယ် context တစ်ခုချင်းအလိုက် ကွဲပြားနေသည် — ဒါသည် DDD ၏ အဓိကသဘောတရားဖြစ်သည်။

---

## ၂.၂ Aggregates, Entities, Value Objects

DDD တွင် domain model ကို မည်သို့ structure ဆောက်မည်ဆိုသည်ကို ဤ building blocks သုံးမျိုးဖြင့် သတ်မှတ်သည်။

### Entity

**Entity** ဆိုသည်မှာ unique identity ရှိသော domain object တစ်ခုဖြစ်သည်။ attributes (ဂုဏ်သတ္တိ) ပြောင်းသော်လည်း identity ပြောင်းလဲမည်မဟုတ်ပါ။

```python
# entity.py

class Order:
    def __init__(self, order_id: str, customer_id: str):
        # order_id သည် identity — ပြောင်းလဲမည်မဟုတ်
        self.order_id = order_id
        self.customer_id = customer_id
        self.status = "PENDING"
        self.items = []

    def add_item(self, item):
        self.items.append(item)

    def confirm(self):
        # Status ပြောင်းသော်လည်း identity (order_id) မပြောင်း
        self.status = "CONFIRMED"

    def __eq__(self, other):
        # Entity equality သည် identity ကို မှီခိုသည်
        return isinstance(other, Order) and self.order_id == other.order_id
```

### Value Object

**Value Object** ဆိုသည်မှာ identity မရှိသော domain object တစ်ခုဖြစ်ပြီး တန်ဖိုး (value) အားဖြင့်သာ ညီမျှမှုကို ဆုံးဖြတ်သည်။ Immutable (ပြောင်းလဲ၍မရ) ဖြစ်သည်။

```python
# value_object.py
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)  # frozen=True သည် immutable ဖြစ်စေသည်
class Money:
    amount: Decimal
    currency: str  # "USD", "MMK", "SGD"

    def add(self, other: 'Money') -> 'Money':
        if self.currency != other.currency:
            raise ValueError("ငွေကြေးမတူ — ပေါင်း၍မရပါ")
        # Immutable: ဟောင်းသော object မပြောင်း၊ အသစ်ဖန်တီးသည်
        return Money(self.amount + other.amount, self.currency)

    def multiply(self, quantity: int) -> 'Money':
        return Money(self.amount * quantity, self.currency)

# Value Object equality သည် value ကို မှီခိုသည်
price1 = Money(Decimal("9.99"), "USD")
price2 = Money(Decimal("9.99"), "USD")
assert price1 == price2  # True — identity မဟုတ်ဘဲ value ဖြင့် ညှိသည်

@dataclass(frozen=True)
class Address:
    street: str
    city: str
    postal_code: str
    country: str
```

### Aggregate

**Aggregate** ဆိုသည်မှာ consistency boundary (ညီညွတ်မှုနယ်ပယ်) ဖြင့် ဘောင်ခတ်ထားသော entities နှင့် value objects ၏ cluster တစ်ခုဖြစ်သည်။ **Aggregate Root** မှတဆင့်သာ access ပြုနိုင်သည်။

```python
# aggregate.py

class OrderItem:  # Entity (within aggregate)
    def __init__(self, product_id: str, quantity: int, price: Money):
        self.product_id = product_id
        self.quantity = quantity
        self.price = price  # Value Object

class Order:  # Aggregate Root
    def __init__(self, order_id: str, customer_id: str):
        self.order_id = order_id
        self.customer_id = customer_id
        self._items = []  # Private — external code တိုက်ရိုက် access မရ
        self._status = "PENDING"
        self._total = Money(Decimal("0"), "USD")

    def add_item(self, product_id: str, quantity: int, unit_price: Money):
        # Aggregate Root မှတဆင့်သာ items ထည့်ခွင့်ရှိသည်
        if self._status != "PENDING":
            raise DomainError("Confirmed order ကို ပြင်၍မရပါ")

        item = OrderItem(product_id, quantity, unit_price)
        self._items.append(item)
        # Aggregate သည် internal consistency ကို ထိန်းသည်
        self._total = self._total.add(unit_price.multiply(quantity))

    def confirm(self):
        if not self._items:
            raise DomainError("Item မပါသော Order ကို confirm မလုပ်နိုင်ပါ")
        self._status = "CONFIRMED"

    @property
    def total(self) -> Money:
        return self._total
```

**Aggregate ဆောက်တည်မှုဆိုင်ရာ Rules**

```
1. Aggregate Root မှတဆင့်သာ Aggregate ကို ဝင်ရောက်ပါ
2. Aggregate တစ်ခုထဲ တစ်ချိန်တည်း transaction တစ်ခုသာ
3. Aggregate ကြားတွင် ID reference (object reference မဟုတ်) သုံးပါ
4. Aggregate ကို သေးငယ်အောင် ထိန်းပါ
```

---

## ၂.၃ Context Mapping

Context Mapping သည် Bounded Context နှစ်ခုသို့မဟုတ် ထိုထက်ပို၍ ဆက်ဆံမှုပုံစံကို သတ်မှတ်သည်။

### Shared Kernel

Contexts နှစ်ခု shared code (model တစ်ဝက်) ကို အတူ ပိုင်ဆိုင်သည်။ ပြောင်းလဲချင်ပါက နှစ်ဖက်ညှိနှိုင်းရသည်။

```
┌──────────────┐     ┌──────────────┐
│  Ordering    │     │  Shipping    │
│  Context     │     │  Context     │
└──────┬───────┘     └──────┬───────┘
       │    ┌──────────┐    │
       └───▶│  Shared  │◀───┘
            │  Kernel  │
            │(Customer │
            │Identity) │
            └──────────┘
```

### Customer-Supplier

**Supplier (upstream)** သည် **Customer (downstream)** ၏ လိုအပ်ချက်ကို ဖြည့်ဆည်းသည်။ Customer သည် Supplier ၏ model ကို adapt လုပ်ရသည်။

```
┌──────────────────┐  API contract    ┌──────────────────┐
│ Product Catalog  │────────────────▶ │  Order Service   │
│ (Supplier /      │                  │  (Customer /     │
│  Upstream)       │                  │  Downstream)     │
└──────────────────┘                  └──────────────────┘
```

### Conformist

Downstream သည် Upstream ၏ model ကို ဆန့်ကျင်ဘဲ လိုက်ညီအောင် လုပ်ရသည်။ (Upstream ၏ power position ကြောင့်)

### Anti-Corruption Layer ပြန်သုံးမည်

Downstream သည် Upstream ၏ model ကို လက်မခံဘဲ translation layer ထောင်သည်။ (Chapter 1 တွင် ဖော်ပြခဲ့ပြီး)

---

## ၂.၄ Strategic DDD မှ Service Boundaries သို့

Bounded Context တစ်ခုသည် Microservice တစ်ခု မဟုတ်ဘဲ၊ Bounded Context တစ်ခုသည် Microservice များစွာ ဖြစ်နိုင်သည် (သို့) Microservice တစ်ခုသည် Bounded Context တစ်ပိုင်းဖြစ်နိုင်သည်။

**Service Boundary သတ်မှတ်ရာတွင် ထည့်သွင်းစဉ်းစားရမည့်အချက်များ**

```
Boundary သတ်မှတ်ချက်
├── Business Cohesion  → Domain logic တူညီသော things တွဲပါ
├── Data Ownership     → Data ကို ဘယ် service က own လုပ်မည်
├── Team Boundary      → Conway's Law — team တစ်ခုစီ service တစ်ခု
├── Change Frequency   → တူညီသောနှုန်းဖြင့် ပြောင်းလဲသော things တွဲပါ
└── Deployment Unit    → တွဲ deploy လုပ်ရသောအရာများ တွဲပါ
```

**E-Commerce မှ ဥပမာ**

```
Domain Analysis → Service Mapping
─────────────────────────────────────────────────────
Bounded Context       → Microservice(s)
─────────────────────────────────────────────────────
Identity/Auth Context → AuthService
                         (JWT issuing, user credentials)

Product Context       → CatalogService
                         (product data, search)
                       + PricingService
                         (dynamic pricing, promotions)

Order Context         → OrderService
                         (order lifecycle)
                       + CartService
                         (session-based cart)

Payment Context       → PaymentService
                         (payment processing)
                       + RefundService
                         (refund workflows)

Fulfillment Context   → InventoryService
                         (stock levels)
                       + ShippingService
                         (carrier integration)
─────────────────────────────────────────────────────
```

**ဆက်သွယ်မှုပုံစံ**

```python
# order_service/domain/events.py

# Domain Event — Service ကြား ဆက်သွယ်ရာတွင် events သုံးသည်
class OrderPlaced:
    def __init__(self, order_id: str, customer_id: str, total: Money, items: list):
        self.order_id = order_id
        self.customer_id = customer_id
        self.total = total
        self.items = items
        self.occurred_at = datetime.utcnow()

# OrderService သည် ဤ event publish လုပ်သည်
# PaymentService, InventoryService, NotificationService တို့ subscribe လုပ်သည်
```

---

## ၂.၅ Event Storming Workshop

**Event Storming** သည် Alberto Brandolini မှ တင်ဆက်သော collaborative workshop technique တစ်ခုဖြစ်ပြီး domain experts (business people) နှင့် developers တို့ ပူးပေါင်းကာ domain events တွေ့ရှိ၍ system ကို model ဆောက်သည့် နည်းလမ်းဖြစ်သည်။

### Event Storming Process

**ကိုင်တွယ်ရာတွင် လိုအပ်သောပစ္စည်းများ**
- Orange Post-it: Domain Events
- Blue Post-it: Commands
- Yellow Post-it: Aggregates
- Purple Post-it: Policies / Reactions
- Pink Post-it: External Systems
- Green Post-it: Read Models

**Big Picture Event Storming (Phase 1)**

```
Timeline →
─────────────────────────────────────────────────────────▶

[Customer     [Product     [Order       [Payment     [Item
 Registered]   Added to     Placed]      Processed]   Shipped]
               Cart]
               
[Cart         [Order       [Payment     [Order
 Created]      Confirmed]   Failed]      Cancelled]

ကြောင်းမှ ၎င်းကို Orange Post-its (events) သည် timeline တစ်လျှောက် paste လုပ်ထားသည်
```

**Process Level Event Storming (Phase 2)**

```
Command          Actor        Aggregate     Event
────────────────────────────────────────────────────────────
[Place Order] → [Customer] → [Order    ] → [Order Placed   ]
[Pay Order  ] → [Customer] → [Payment  ] → [Payment Success]
[Ship Items ] → [Warehouse] → [Shipment ] → [Items Shipped  ]
[Cancel Order]→ [Customer] → [Order    ] → [Order Cancelled]
```

**Policy (Reaction) Examples**

```
WHEN [Order Placed]
THEN [Reserve Inventory]  ← Policy (InventoryService)

WHEN [Payment Failed]
THEN [Cancel Order]       ← Policy (OrderService)

WHEN [Order Shipped]
THEN [Send Notification]  ← Policy (NotificationService)
```

**Event Storming Output → Service Boundaries**

Workshop ပြီးနောက် sticky notes cluster ကြည့်ကာ natural grouping (သဘာဝအုပ်စု) တွေ့ပါ —

```
Cluster Analysis
├── Customer/Identity notes  → AuthService + UserService
├── Catalog/Cart notes       → CatalogService + CartService
├── Order notes              → OrderService
├── Payment notes            → PaymentService
└── Fulfillment notes        → InventoryService + ShippingService
```

**Event Storming Code Template**

```python
# event_storming_output/order_aggregate.py

from datetime import datetime
from typing import List
from decimal import Decimal

class OrderItem:
    def __init__(self, product_id: str, quantity: int, unit_price: Decimal):
        self.product_id = product_id
        self.quantity = quantity
        self.unit_price = unit_price

class Order:  # Event Storming မှ ထွက်ပေါ်လာသော Aggregate
    def __init__(self, order_id: str, customer_id: str):
        self.order_id = order_id
        self.customer_id = customer_id
        self._items: List[OrderItem] = []
        self._status = "DRAFT"
        self._events = []  # Domain events collect လုပ်သည်

    def place(self, items: List[OrderItem]):
        # Command: Place Order
        if self._status != "DRAFT":
            raise ValueError("ဤ order ကို place လုပ်၍မရပါ")
        self._items = items
        self._status = "PLACED"
        # Domain Event ထုတ်သည်
        self._events.append(OrderPlaced(
            order_id=self.order_id,
            customer_id=self.customer_id,
            items=items,
            placed_at=datetime.utcnow()
        ))

    def cancel(self, reason: str):
        # Command: Cancel Order
        if self._status not in ("PLACED", "CONFIRMED"):
            raise ValueError("ဤ status တွင် cancel မလုပ်နိုင်ပါ")
        self._status = "CANCELLED"
        self._events.append(OrderCancelled(
            order_id=self.order_id,
            reason=reason,
            cancelled_at=datetime.utcnow()
        ))

    def pull_events(self) -> list:
        # Events များကို ထုတ်ပြီး clear လုပ်သည်
        events = self._events.copy()
        self._events.clear()
        return events
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **Ubiquitous Language** — Business expert နှင့် developer တူညီသောဘာသာ (language) သုံးပါ၊ code တွင် domain terms ကို ရောင်ပြန်ဟပ်ပါ။
2. **Bounded Context** — Domain model တစ်ခု valid ဖြစ်သည့် explicit boundary ကို သတ်မှတ်ပါ။ "Product" ၏ အဓိပ္ပာယ် context တိုင်း ကွဲနိုင်သည်။
3. **Aggregate** — Consistency boundary ဖြင့် ဘောင်ခတ်ထားသော entities/value objects cluster ဖြစ်ပြီး Aggregate Root မှတဆင့်သာ ဝင်ရောက်နိုင်သည်။
4. **Entity** သည် identity ဖြင့် ကိုယ်ထူကိုယ်ထ ရပ်တည်ပြီး **Value Object** သည် value ဖြင့်သာ ညီမျှမှု ဆုံးဖြတ်သည်၊ immutable ဖြစ်သင့်သည်။
5. **Context Mapping** — Shared Kernel, Customer-Supplier, Conformist, ACL — မည်သည့် relationship pattern ဖြင့် contexts ဆက်ဆံမည်ကို ရှင်းရှင်းလင်းလင်း သတ်မှတ်ပါ။
6. **Event Storming** သည် domain experts နှင့် developers ပူးပေါင်း domain ကို explore လုပ်ကာ service boundaries ကို ရှာဖွေသည့် collaborative technique ဖြစ်သည်။
7. Bounded Context တစ်ခုနှင့် Microservice တစ်ခု **အမြဲ one-to-one မဟုတ်** — granularity ကို team size, deployment needs, data ownership တို့ပေါ် မူတည်၍ ဆုံးဖြတ်ပါ။
