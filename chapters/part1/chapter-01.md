# အခန်း ၁ — Monolith မှ Microservices သို့

## နိဒါန်း

Software ဖွံ့ဖြိုးတိုးတက်မှုသမိုင်းတွင် application များကို တည်ဆောက်ပုံသည် နှစ်ပေါင်းများစွာကြာ Monolithic Architecture ပေါ်တွင် မှီခိုနေခဲ့သည်။ သို့သော် digital transformation ဟောင်းကြီးတိုးချဲ့လာသောအခါ၊ user အရေအတွက်တိုးလာသောအခါ၊ နှင့် feature များပိုမိုရှုပ်ထွေးလာသောအခါ — Monolith တစ်ခုတည်းဖြင့် ဖြေရှင်းရန် ခက်ခဲလာသည်ဟု developer များ တွေ့ကြုံလာကြသည်။ ထို challenge များကို ဖြေရှင်းနိုင်ရန် Microservices Architecture ပေါ်ထွန်းလာခဲ့သည်။ ဤအခန်းတွင် Microservices ဆိုတာဘာလဲဆိုသည့် အခြေခံမှ စတင်ကာ Monolith မှ Microservices သို့ ကူးပြောင်းရာတွင် အသုံးပြုသော နည်းဗျူဟာများ၊ ပုံစံများ (patterns) နှင့် ကြုံတွေ့နိုင်သည့် ပြဿနာများကို အသေးစိတ် လေ့လာသွားမည်ဖြစ်သည်။

---

## ၁.၁ Microservice ဆိုတာဘာလဲ

**Microservice** ဆိုသည်မှာ တစ်ခုတည်းသော business capability (လုပ်ငန်းစွမ်းရည်) တစ်ခုကို ဆောင်ရွက်ပေးသည့် သေးငယ်၍ လွတ်လပ်စွာ ဖြန့်ချိနိုင်သော (independently deployable) service တစ်ခုဖြစ်သည်။

**သဘောတရားအဓိပ္ပာယ်** —
- **Single Responsibility**: Service တစ်ခုစီသည် တစ်ခုတည်းသော တာဝန်ကိုသာ ဆောင်ရွက်သည်
- **Loose Coupling**: Services အချင်းချင်း ပေါင်းကူးမှု နည်းပါးသည်
- **High Cohesion**: Service တစ်ခုအတွင်း ဆက်နွှယ်မှု မြင့်မားသည်
- **Independent Deployment**: Service တစ်ခုကို ကျန် services များကို မထိခိုက်ဘဲ deploy လုပ်နိုင်သည်

**Monolith နှင့် Microservices ကွာခြားချက်**

```
┌─────────────────────────────────────────────────────┐
│           MONOLITHIC APPLICATION                    │
│  ┌──────────┬──────────┬──────────┬──────────────┐  │
│  │   User   │  Order   │ Payment  │  Inventory   │  │
│  │  Module  │  Module  │  Module  │   Module     │  │
│  └──────────┴──────────┴──────────┴──────────────┘  │
│              Single Database                        │
│              Single Deploy Unit                     │
└─────────────────────────────────────────────────────┘

              ↓  Microservices ပုံစံ

┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  User    │  │  Order   │  │ Payment  │  │Inventory │
│ Service  │  │ Service  │  │ Service  │  │ Service  │
│   DB     │  │   DB     │  │   DB     │  │   DB     │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
    ↑              ↑              ↑              ↑
 Port 8001      Port 8002      Port 8003      Port 8004
```

Microservice တစ်ခုစီတွင် —
- မိမိပိုင် database ရှိသည် (Database per Service pattern)
- မိမိပိုင် codebase ရှိသည်
- မိမိ lifecycle ဖြင့် deploy လုပ်နိုင်သည်
- Network မှတဆင့် (HTTP, gRPC, messaging) ဆက်သွယ်သည်

---

## ၁.၂ ဘယ်အချိန်မှာ Split လုပ်မလဲ

Monolith တစ်ခုကို Microservices သို့ ချက်ချင်းပြောင်းလဲသင့်သည်ဟု မဆိုလိုပါ။ ဤ decision ကို မှန်ကန်စွာ ချနိုင်ရန် အောက်ပါ signal များကို ကြည့်ရမည်။

**Split လုပ်သင့်သည့် သတိပေး လက္ခဏာများ (Warning Signs)**

| လက္ခဏာ | ရှင်းလင်းချက် |
|---------|--------------|
| **Deployment bottleneck** | Feature တစ်ခုအတွက် full app ကို redeploy လုပ်ရသည် |
| **Team scaling issues** | Developer များ တစ်ဦးနှင့်တစ်ဦး codebase တွင် ပဋိပက္ခဖြစ်သည် |
| **Scaling မညီမျှ** | Module တစ်ခုသာ high traffic ဖြစ်ပြီး ကျန်သည် idle ဖြစ်နေသည် |
| **Technology lock-in** | တစ်ချို့ module အတွက် different tech stack လိုအပ်သည် |
| **Build & test slow** | CI/CD pipeline ကြာမြင့်သည် |
| **Reliability cascades** | Module တစ်ခု fail ဖြစ်လျှင် entire app ကျသည် |

**Split မလုပ်သင့်သည့် အချိန်**

- Team size သေးငယ်သောအခါ (2-pizza rule မဖြစ်သေးပါက)
- Product-market fit မရသေးသောအခါ
- Business logic မတည်ငြိမ်သေးသောအခါ
- Distributed systems experience မရှိသောအခါ

Martin Fowler က "Monolith First" ဟူသော သဘောတရားကို တင်ပြသည် — application ကို Monolith ဖြင့် စတင်ပြီး complexity ပြသောအခါတွင်မှ split လုပ်ရန် အကြံပေးသည်။

---

## ၁.၃ Decomposition Strategies

Monolith ကို Microservices အဖြစ် ဘယ်လိုပိုင်းဖြတ်မည်ဆိုသည်မှာ အရေးကြီးဆုံး decision တစ်ခုဖြစ်သည်။ နည်းလမ်းသုံးမျိုး ရှိသည်။

### ၁.၃.၁ Decompose by Business Capability

Business capability ဆိုသည်မှာ organization တစ်ခုလုပ်ဆောင်သည့် လုပ်ငန်းအမျိုးအစားဖြစ်သည်။

```
E-Commerce Application
├── Customer Management   → CustomerService
├── Order Management      → OrderService
├── Payment Processing    → PaymentService
├── Inventory Management  → InventoryService
├── Shipping & Delivery   → ShippingService
└── Notification          → NotificationService
```

**ဥပမာ — Order Service**

```python
# order_service.py
class OrderService:
    def __init__(self, order_repo, payment_client, inventory_client):
        self.order_repo = order_repo
        # payment_client နှင့် inventory_client သည် external services
        self.payment_client = payment_client
        self.inventory_client = inventory_client

    def create_order(self, user_id, items):
        # ကုန်ပစ္စည်း availability စစ်ဆေးသည်
        for item in items:
            if not self.inventory_client.check_availability(item.product_id, item.quantity):
                raise InsufficientStockError(item.product_id)

        # Order ဖန်တီးသည်
        order = Order(user_id=user_id, items=items, status="PENDING")
        return self.order_repo.save(order)
```

### ၁.၃.၂ Decompose by Domain (Domain-Driven Design)

Domain-Driven Design (DDD) ၏ Bounded Context concept ကို အသုံးပြု၍ service boundaries သတ်မှတ်သည်။

```
Domain: E-Commerce
├── Catalog Domain     → ProductCatalogService
├── Ordering Domain    → OrderService
├── Billing Domain     → BillingService
└── Identity Domain    → AuthService
```

Service boundary ကို domain expert နှင့် developer တို့ပူးပေါင်း Event Storming session မှ ထွက်ပေါ်လာသည့် Bounded Context များအတိုင်း သတ်မှတ်သည်။

### ၁.၃.၃ Decompose by Volatility (ပြောင်းလဲနှုန်း)

မည်သည့် functionality သည် မကြာခဏ ပြောင်းလဲသနည်း ဟု ဆန်းစစ်ပြီး ထို volatile (မတည်ငြိမ်သော) part ကို separate service အဖြစ် ခွဲထုတ်သည်။

```
High Volatility  → Pricing Engine     (promotion rules မကြာခဏ ပြောင်းသည်)
Medium Volatility → Order Processing  (business rules တည်ငြိမ်သည်)
Low Volatility   → User Authentication (မကြာခဏ မပြောင်းပါ)
```

---

## ၁.၄ Strangler Fig Pattern

**Strangler Fig Pattern** သည် Monolith ကို တစ်ချက်ချင်း replace မလုပ်ဘဲ တဖြည်းဖြည်း microservices အဖြစ် ပြောင်းလဲသည့် migration strategy တစ်ခုဖြစ်သည်။ သစ်ပင်ကို ဖုံးအုပ်ပြီး တဖြည်းဖြည်း အစားထိုးသည့် Strangler Fig သစ်ပင် (Ficus macrocarpa) မှ နာမည်ရသည်။

```
Phase 1: Proxy ချိတ်ဆက်မှု
┌─────────┐     ┌─────────────┐     ┌──────────────┐
│ Client  │────▶│   Proxy /   │────▶│   Monolith   │
│         │     │ API Gateway │     │ (existing)   │
└─────────┘     └─────────────┘     └──────────────┘

Phase 2: Feature တစ်ခု extract လုပ်
┌─────────┐     ┌─────────────┐     ┌──────────────┐
│ Client  │────▶│   Proxy /   │────▶│   Monolith   │
│         │     │ API Gateway │  └─▶│ (shrinking)  │
└─────────┘     └─────────────┘  │  └──────────────┘
                                 │  ┌──────────────┐
                                 └─▶│  New Service │
                                    │  (extracted) │
                                    └──────────────┘

Phase 3: Monolith ပြီးစီးသောအခါ
┌─────────┐     ┌─────────────┐     ┌──────────────┐
│ Client  │────▶│ API Gateway │────▶│  Service A   │
│         │     │             │────▶│  Service B   │
└─────────┘     └─────────────┘────▶│  Service C   │
                                    └──────────────┘
```

**Implementation နမူနာ**

```python
# api_gateway.py
import httpx

class StranglerFigGateway:
    def __init__(self):
        # Route map - extracted services ကို သတ်မှတ်သည်
        self.extracted_routes = {
            "/api/orders": "http://order-service:8001",
            "/api/payments": "http://payment-service:8002",
        }
        # ကျန် routes သည် monolith သို့ ဦးတည်သည်
        self.monolith_url = "http://monolith:8000"

    async def route_request(self, path, method, body):
        # Extracted route ကို စစ်ဆေးသည်
        for route_prefix, service_url in self.extracted_routes.items():
            if path.startswith(route_prefix):
                return await self.forward_to(service_url, path, method, body)

        # Default: monolith သို့ forward လုပ်သည်
        return await self.forward_to(self.monolith_url, path, method, body)

    async def forward_to(self, base_url, path, method, body):
        async with httpx.AsyncClient() as client:
            response = await client.request(method, f"{base_url}{path}", json=body)
            return response
```

---

## ၁.၅ Anti-Corruption Layer (ACL)

**Anti-Corruption Layer** သည် Bounded Context နှစ်ခုကြား (ဥပမာ — Legacy Monolith နှင့် New Microservice ကြား) translation layer တစ်ခုဖြစ်သည်။ Legacy system ၏ domain model သည် new system ကို "ညစ်ညမ်းစေ" (corrupt) ခြင်းမှ ကာကွယ်သည်။

```
┌──────────────────┐     ┌──────────────────────┐     ┌───────────────────┐
│   New Service    │────▶│ Anti-Corruption Layer│────▶│  Legacy Monolith  │
│  (clean domain)  │◀────│   (translation)      │◀────│  (old domain)     │
└──────────────────┘     └──────────────────────┘     └───────────────────┘
```

**ACL Code နမူနာ**

```python
# anti_corruption_layer.py

# Legacy Monolith မှ ဟောင်းနွမ်းသော data format
class LegacyCustomerDTO:
    def __init__(self, cust_id, f_name, l_name, ph_no):
        self.cust_id = cust_id    # Legacy field name
        self.f_name = f_name
        self.l_name = l_name
        self.ph_no = ph_no        # Legacy phone format

# New Service ၏ clean domain model
class Customer:
    def __init__(self, customer_id, full_name, phone_number):
        self.customer_id = customer_id
        self.full_name = full_name
        self.phone_number = phone_number

# ACL - Translation logic
class CustomerAntiCorruptionLayer:
    def translate_from_legacy(self, legacy_dto: LegacyCustomerDTO) -> Customer:
        # Legacy format ကို clean domain model သို့ ပြောင်းသည်
        return Customer(
            customer_id=str(legacy_dto.cust_id),
            full_name=f"{legacy_dto.f_name} {legacy_dto.l_name}",
            phone_number=self.normalize_phone(legacy_dto.ph_no)
        )

    def normalize_phone(self, legacy_phone: str) -> str:
        # ဟောင်းသော phone format ကို standard format သို့ ပြောင်းသည်
        cleaned = legacy_phone.replace("-", "").replace(" ", "")
        return f"+95{cleaned}" if not cleaned.startswith("+") else cleaned
```

---

## ၁.၆ ဖြစ်နိုင်သောပြဿနာများ — Complexity vs Scalability vs Team Autonomy

Microservices သည် မည်သည့် silver bullet မဟုတ်ပါ။ အောက်ပါ trade-off များကို နားလည်ရမည်။

### Complexity (ရှုပ်ထွေးမှု)

**Monolith** — function call ဖြင့် ဆက်သွယ်သောကြောင့် debugging လွယ်ကူသည်
**Microservices** — Network calls, distributed tracing, service discovery, load balancing လိုအပ်သည်

```
ပြဿနာများ —
├── Distributed transactions (Saga pattern လိုသည်)
├── Network latency & failures
├── Data consistency (eventual consistency)
├── Service discovery complexity
├── Observability (distributed tracing လိုသည်)
└── Operational overhead (Kubernetes, Docker, etc.)
```

### Scalability (ချဲ့ထွင်နိုင်မှု)

**Microservices ၏ scaling အကျိုးကျေးဇူး**

```
Before (Monolith):
┌────────────────────────────────────┐  × 5 instances
│  User + Order + Payment + Catalog  │
└────────────────────────────────────┘
Resource waste: Payment module ကို scale လုပ်ချင်သော်
                ကျန် modules များပါ scale လုပ်ရသည်

After (Microservices):
┌──────────┐ × 1    ┌──────────┐ × 10   ┌──────────┐ × 2
│  User    │        │ Payment  │        │  Order   │
│ Service  │        │ Service  │        │ Service  │
└──────────┘        └──────────┘        └──────────┘
Resource efficient: Payment service ကိုသာ scale လုပ်နိုင်သည်
```

### Team Autonomy (အဖွဲ့လွတ်လပ်မှု)

Conway's Law — "Organizations design systems that mirror their own communication structure"

```
Microservices + Team Structure
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Team Alpha    │  │   Team Beta     │  │   Team Gamma    │
│ OrderService    │  │ PaymentService  │  │ UserService     │
│ - Own CI/CD     │  │ - Own CI/CD     │  │ - Own CI/CD     │
│ - Own DB        │  │ - Own DB        │  │ - Own DB        │
│ - Own tech stack│  │ - Own tech stack│  │ - Own tech stack│
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **Microservice** ဆိုသည်မှာ single business capability ကိုသာ တာဝန်ယူသော independently deployable service တစ်ခုဖြစ်သည်။
2. **Split ချိန်** — scaling bottleneck, team coordination issues, deployment friction တို့ ပေါ်မှသာ Microservices သို့ ကူးပြောင်းပါ။ "Monolith First" သဘောတရားကို လိုက်နာပါ။
3. **Decomposition Strategies** သုံးမျိုး — Business Capability, Domain (DDD), Volatility တို့ကို context အလိုက် ရွေးချယ်ပါ။
4. **Strangler Fig Pattern** သည် Monolith ကို တဖြည်းဖြည်း replace လုပ်သည့် low-risk migration strategy ဖြစ်သည်။
5. **Anti-Corruption Layer** သည် Legacy system ၏ domain model ညစ်ညမ်းမှုမှ new service ကို ကာကွယ်သည်။
6. Microservices ၏ **trade-offs** — operational complexity တိုးသည်၊ scalability နှင့် team autonomy ကောင်းလာသည်။
