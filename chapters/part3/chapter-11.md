# အခန်း ၁၁ — CQRS — Command Query Responsibility Segregation

## နိဒါန်း

ရိုးရာ application architecture တွင် read နှင့် write operations နှစ်ခုလုံးကို တူညီသော data model ဖြင့် handle လုပ်သည်။ Database table တစ်ခုဒီဇိုင်းကို write (normalization) နှင့် read (join queries) နှစ်ခုလုံးအတွက် အလုပ်လုပ်အောင် ကြိုးစားရသည်။ ဤနှစ်ဦးနှစ်ဖက် ကျေနပ်ချေ optimized မဟုတ်သော compromise သည် ကောင်းမွန်သော scalability ကို ပိတ်ဆို့ထားသည်။ **CQRS (Command Query Responsibility Segregation)** သည် read model နှင့် write model ကို လုံးဝ ခွဲထုတ်သဖြင့် တစ်ခုချင်းစီကို သီးသန့် optimize ပြုလုပ်နိုင်သည်။ ဤအခန်းတွင် CQRS ၏ core concepts၊ projections၊ eventual consistency trade-offs နှင့် practical implementation patterns များကို အသေးစိတ် ဆွေးနွေးမည်။

---

## ၁၁.၁ Separating Read and Write Models

### Traditional (Single Model) Approach

```
Client ──→ OrderService ──→ [orders table]
              (read query)         ↕
              (write command)   (same table)
```

ပြဿနာ — orders table ကို complex join query နှင့် normalize ထားသော write logic နှစ်ခုလုံးအတွက် ကြိုးစားရသည်။

### CQRS Approach

```
Write Side:
Client ──→ CommandHandler ──→ Aggregate (domain logic)
                          ──→ [Write DB (normalized)]
                          ──→ Events ─────────────────────┐
                                                           ↓
Read Side:                                         Projection Handler
Client ──→ QueryHandler  ──→ [Read DB (denormalized)] ←───┘
                              (fast, pre-joined views)
```

### Write Model (Command Side)

Write model သည် business rules နှင့် domain logic ကို ကိုင်တွယ်သည်။ Normalized ဖြစ်ပြီး ACID properties ကို လိုက်နာသည်။

```python
# Command Handler — write side
class PlaceOrderCommandHandler:

    def handle(self, command: PlaceOrderCommand):
        # ၁။ aggregate load လုပ်သည် (read from write DB)
        customer = self.customer_repo.get(command.customer_id)
        products = self.product_repo.get_many(command.product_ids)

        # ၂။ domain validation လုပ်သည်
        if not customer.is_active():
            raise CustomerNotActiveError()      # inactive ဖောက်သည်
        if not self._all_products_available(products):
            raise ProductNotAvailableError()    # product မရှိ

        # ၃။ aggregate ဖန်တီး၍ business logic ဆောင်ရွက်သည်
        order = Order.create(
            customer_id=command.customer_id,
            items=command.items,
            total=self._calculate_total(products, command.items)
        )

        # ၄။ write DB ထဲ save လုပ်သည်
        self.order_repo.save(order)

        # ၅။ integration event publish လုပ်သည်
        self.event_bus.publish(OrderPlacedEvent(
            order_id=order.id,
            customer_id=order.customer_id,
            total=order.total
        ))
```

### Read Model (Query Side)

Read model သည် query performance အတွက် optimized ဖြစ်ပြီး denormalized view ပုံစံဖြင့် ထားသည်။

```python
# Query Handler — read side (fast reads)
class GetOrderSummaryQueryHandler:

    def handle(self, query: GetOrderSummaryQuery):
        # pre-joined, denormalized table မှ တိုက်ရိုက် query
        # no joins needed — very fast
        row = self.read_db.query_one("""
            SELECT
                o.order_id,
                o.customer_name,    -- denormalized
                o.customer_email,   -- denormalized
                o.total_amount,
                o.status,
                o.item_count,       -- pre-calculated
                o.created_at
            FROM order_summaries o  -- denormalized read table
            WHERE o.order_id = %s
        """, (query.order_id,))

        return OrderSummaryDto.from_row(row)
```

---

## ၁၁.၂ Read Model Projections

**Projection** သည် event stream ကို subscribe ပြီး read model ကို update လုပ်သော component ဖြစ်သည်။

```python
# Order Projection — events မှ read model တည်ဆောက်သည်
class OrderProjection:

    @event_handler('OrderPlaced')
    def on_order_placed(self, event):
        """အမှာစာ တင်သောအခါ read model ထဲ insert"""
        self.read_db.execute("""
            INSERT INTO order_summaries (
                order_id, customer_id, customer_name,
                total_amount, status, item_count, created_at
            ) VALUES (%s, %s, %s, %s, %s, %s, %s)
        """, (
            event.data['orderId'],
            event.data['customerId'],
            event.data['customerName'],  # denormalized customer name
            event.data['totalAmount'],
            'pending',                   # initial status
            len(event.data['items']),    # item count pre-calculated
            event.occurred_at
        ))

    @event_handler('OrderShipped')
    def on_order_shipped(self, event):
        """အမှာစာ ပို့သောအခါ read model update"""
        self.read_db.execute("""
            UPDATE order_summaries
            SET status = 'shipped',
                tracking_number = %s,   -- tracking number ထည့်သည်
                shipped_at = %s
            WHERE order_id = %s
        """, (
            event.data['trackingNumber'],
            event.occurred_at,
            event.data['orderId']
        ))

    @event_handler('OrderCancelled')
    def on_order_cancelled(self, event):
        """အမှာစာ ပယ်ဖျက်သောအခါ status update"""
        self.read_db.execute("""
            UPDATE order_summaries
            SET status = 'cancelled',
                cancelled_reason = %s,   -- ပယ်ဖျက်သောအကြောင်း
                cancelled_at = %s
            WHERE order_id = %s
        """, (
            event.data.get('reason', 'unknown'),
            event.occurred_at,
            event.data['orderId']
        ))
```

### Multiple Projections from Same Events

တူညီသော event stream မှ မတူညီသော read models များကို build နိုင်သည်။

```
OrderPlaced Event
        │
        ├──→ OrderSummaryProjection   ──→ [order_summaries table]
        ├──→ CustomerOrdersProjection ──→ [customer_orders table]
        ├──→ DashboardProjection      ──→ [daily_stats table]
        └──→ SearchIndexProjection    ──→ [Elasticsearch index]
```

---

## ၁၁.၃ Eventual Consistency Trade-offs

CQRS system တွင် command ကို handle ပြီးနောက် read model သည် ချက်ချင်း update မဖြစ်ဘဲ နည်းနည်းနောက်ကျနိုင်သည်။ ဤ lag ကို **eventual consistency** ဟုခေါ်သည်။

```
Timeline:
T+0ms:  User places order (command processed, write DB updated)
T+5ms:  OrderPlaced event published to message broker
T+50ms: Projection handler receives event, updates read DB
T+51ms: User queries order status → sees updated state ✅

                 ↑
           0ms - 50ms = consistency lag (eventual)
```

### ပြဿနာ နှင့် ဖြေရှင်းနည်း

```python
# ပြဿနာ: command ပြီးနောက် ချက်ချင်း query ဖြစ်ချင်သောအခါ
def place_order_and_redirect(request):
    command = PlaceOrderCommand(...)
    order_id = command_bus.send(command)    # command send

    # BUG: read model မ update ရသေးသောကြောင့် 404 ရနိုင်
    return redirect(f'/orders/{order_id}')

# ဖြေရှင်းနည်း ၁: Client-side polling
def place_order_with_polling(request):
    command = PlaceOrderCommand(...)
    order_id = command_bus.send(command)
    # Frontend တွင် retry logic ထည့်ပါ
    return {'orderId': order_id, 'status': 'processing'}

# ဖြေရှင်းနည်း ၂: Optimistic UI — assume success
# ဖြေရှင်းနည်း ၃: Command result ကို response မှ ပြန်ပေးသည်
```

---

## ၁၁.၄ Materialized Views & Denormalization

**Materialized View** သည် pre-computed query result ကို သိမ်းဆည်းသောအရာဖြစ်သည်။ CQRS ၏ read model သည် materialized view concept နှင့် ဆင်တူသည်။

### Denormalization ဥပမာ

```sql
-- Write Side (normalized tables):
orders      (order_id, customer_id, total, status, created_at)
customers   (customer_id, name, email, phone)
order_items (item_id, order_id, product_id, qty, price)
products    (product_id, name, sku, category)

-- Read Side (denormalized view for order list page):
-- join မလိုဘဲ single query ဖြင့် ရသည်
CREATE TABLE order_list_view (
    order_id        UUID,
    customer_name   VARCHAR,    -- customers မှ denormalize
    customer_email  VARCHAR,    -- customers မှ denormalize
    total_amount    DECIMAL,
    status          VARCHAR,
    item_count      INT,        -- pre-calculated
    item_names      TEXT[],     -- products မှ denormalize (array)
    created_at      TIMESTAMPTZ,
    -- full-text search index
    search_vector   TSVECTOR
);
```

### ကောင်းသောချက်

```
Traditional normalized query:
SELECT o.*, c.name, c.email,
       COUNT(i.item_id) as item_count,
       ARRAY_AGG(p.name) as items
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items i ON i.order_id = o.id
JOIN products p ON p.id = i.product_id
GROUP BY o.id, c.name, c.email
→ 4 tables join, slow with large data

CQRS denormalized read:
SELECT * FROM order_list_view
→ Single table scan, very fast ✅
```

---

## ၁၁.၅ Synchronizing Read Store from Events

Event stream မှ read store ကို sync လုပ်သောအခါ **checkpoint** (offset tracking) အရေးကြီးသည်။

```python
# Event-driven projection synchronization
class ProjectionSynchronizer:

    def __init__(self, projection, event_store):
        self.projection = projection
        self.event_store = event_store
        # last processed offset ကို database တွင် သိမ်းသည်
        self.checkpoint = self._load_checkpoint()

    def run(self):
        """Continuous event processing loop"""
        while True:
            # checkpoint မှ ဆက်ပြီး event batch ဆွဲယူ
            events = self.event_store.get_events_after(
                self.checkpoint,
                batch_size=100             # batch ဖြင့် process
            )

            if not events:
                time.sleep(0.1)            # event မရှိလျှင် နည်းနည်းစောင့်
                continue

            for event in events:
                try:
                    self.projection.handle(event)   # read model update
                    self.checkpoint = event.global_sequence  # checkpoint update
                    self._save_checkpoint()         # checkpoint persist
                except Exception as e:
                    self._handle_error(event, e)    # error handling
                    break

    def rebuild(self):
        """
        Read model ကို scratch မှ ပြန် build လုပ်သည်
        bug fix ပြီးနောက် သို့မဟုတ် schema change သောအခါ
        """
        self.read_db.truncate_all_projection_tables()  # clear
        self.checkpoint = 0                            # reset to beginning
        self.run()                                     # replay all events
```

---

## ၁၁.၆ CQRS Without Event Sourcing

CQRS ကို Event Sourcing မပါဘဲ သုံးနိုင်သည်။ Write DB တွင် standard tables ထားပြီး read DB ကို separately sync ဖြစ်နိုင်သည်။

```
Write DB (PostgreSQL normalized):
┌──────────────────────────────────────────┐
│  orders, customers, products, inventory  │
└──────────────────────────────────────────┘
          │ DB change triggers / CDC
          ↓
Read DB (Redis / Elasticsearch / MongoDB):
┌──────────────────────────────────────────┐
│  order_views, product_search_index,      │
│  customer_dashboards (pre-joined)        │
└──────────────────────────────────────────┘
```

```python
# Simple CQRS without Event Sourcing
class OrderCommandService:
    """Write side — domain logic"""

    def place_order(self, customer_id, items):
        with self.write_db.transaction():
            order = Order.create(customer_id, items)
            self.write_db.save(order)       # write DB update

        # read store ကို async update (non-blocking)
        self.read_store_updater.update_order(order)
        return order.id

class OrderQueryService:
    """Read side — fast queries"""

    def get_order(self, order_id):
        # read store (Redis cache or read replica) မှ ဆွဲသည်
        cached = self.cache.get(f'order:{order_id}')
        if cached:
            return cached

        # cache miss → read DB query
        return self.read_db.query_order(order_id)

    def search_orders(self, filters):
        # Elasticsearch ကဲ့သို့ search-optimized store မှ query
        return self.search_index.search(filters)
```

### CQRS Complexity Decision Matrix

```
┌────────────────────────────────┬────────────────┬──────────────┐
│ Scenario                       │ CQRS needed?   │ Notes        │
├────────────────────────────────┼────────────────┼──────────────┤
│ Simple CRUD app                │ ❌ Overkill    │ Too complex  │
│ Read >> Write (10:1 ratio)     │ ✅ Consider    │ Scale reads  │
│ Complex domain + reporting     │ ✅ Good fit    │ Separate     │
│ Multiple read formats needed   │ ✅ Good fit    │ Projections  │
│ Real-time analytics            │ ✅ Good fit    │ Stream model │
│ Small team, tight deadline     │ ❌ Start simple│ Add later    │
└────────────────────────────────┴────────────────┴──────────────┘
```

---

## အဓိကသင်ခန်းစာများ

- **CQRS** သည် read model နှင့် write model ကို ခွဲထုတ်သဖြင့် တစ်ခုချင်းစီကို သီးသန့် optimize ပြုလုပ်နိုင်သည်။
- **Write Model** သည် business rules နှင့် domain logic ကို ကိုင်တွယ်ပြီး normalized ဖြစ်သည်။
- **Read Model** သည် query performance အတွက် denormalized ဖြစ်ပြီး projection handler ဖြင့် event မှ update ဖြစ်သည်။
- တူညီသော event stream မှ **multiple projections** ကို build နိုင်ပြီး မတူညီသော read stores (SQL, Redis, Elasticsearch) တွင် သိမ်းနိုင်သည်။
- **Eventual consistency** ကို accept လုပ်ရမည်၊ command ပြီးနောက် read model သည် milliseconds မှ seconds အတွင်း update ဖြစ်မည်ဆိုသည်ကို UX ဒီဇိုင်းတွင် ထည့်တွက်ရမည်။
- **Projection rebuild** (replay) feature သည် bug fix ပြီးနောက် read model ကို ပြန်တည်ဆောက်ရန် ဖြစ်နိုင်သောကြောင့် checkpoint-based sync ကို ဒီဇိုင်းချပါ။
- CQRS ကို Event Sourcing မပါဘဲ သုံးနိုင်ပြီး simple app တွင် မလိုသောကြောင့် complexity ကို ဦးစွာ ဆန်းစစ်ပါ။
