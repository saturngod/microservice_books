# အခန်း ၁၃ — Transactional Outbox & Change Data Capture

## နိဒါန်း

Microservices system တွင် ဖြစ်မကြာခဏ ကြုံတွေ့ရသော ပြဿနာတစ်ခုမှာ — database ကို update လုပ်ပြီး message broker ကို publish လုပ်ရသောအချိန်တွင် တစ်ခုသာ အောင်မြင်ပြီး တစ်ခု မအောင်မြင်သောအခါ data inconsistency ဖြစ်ပေါ်ခြင်းဖြစ်သည်။ ဤ **dual-write problem** ကို ဖြေရှင်းရန် **Transactional Outbox Pattern** သည် အကောင်းဆုံးနည်းလမ်းများထဲမှ တစ်ခုဖြစ်သည်။ ဤပုံစံနှင့်အတူ **Change Data Capture (CDC)** technology ဖြစ်သော Debezium၊ Inbox Pattern နှင့် Schema Registry တို့ကို ဤအခန်းတွင် အသေးစိတ် ဆွေးနွေးမည်။

---

## ၁၃.၁ The Dual-Write Problem

**Dual-Write** ဆိုသည်မှာ code ထဲတွင် database write နှင့် message publish ကို သီးခြားသီးခြား ဆောင်ရွက်ခြင်းဖြစ်ပြီး ၎င်းနှစ်ခုကြားတွင် failure ဖြစ်ပေါ်နိုင်သည်။

```python
# ❌ WRONG — Dual-write ပြဿနာ ဖြစ်သော code
class OrderService:

    def place_order(self, command):
        # Step 1: Database ထဲ save
        order = Order.create(command)
        self.db.save(order)       # ✅ DB save အောင်မြင်

        # CRASH HERE! ← application crash ဖြစ်ရင် message မပို့ရ
        # သို့မဟုတ် message broker down ဖြစ်ရင် message မပို့ရ

        # Step 2: Message publish
        self.message_bus.publish(OrderPlacedEvent(order))  # ❌ fail!
        # ကုသနည်း မရှိသောကြောင့် order DB မှာ ရှိ၊ event မပါ
```

### ဖြစ်ပေါ်နိုင်သော scenarios

```
Scenario A: DB Save OK, Publish Fail
──────────────────────────────────────
DB: order ရှိ ✅
Event Bus: event မရောက် ❌
Result: Inventory မ reserve, Payment မ charge → Ghost order!

Scenario B: Publish OK, DB Save Fail
──────────────────────────────────────
DB: order မရှိ ❌
Event Bus: event ရောက် ✅
Result: Inventory က reserve လုပ်သော်လည်း order မရှိ → Phantom event!
```

---

## ၁၃.၂ Transactional Outbox Pattern

**Transactional Outbox** သည် message ကို directly broker ထဲ မပို့ဘဲ database ၏ **outbox table** ထဲ တူညီသော transaction မှာပင် save လုပ်သည်။ ထို့နောက် သီးခြား process တစ်ခုက outbox table ကို ဖတ်ပြီး broker ထဲ publish လုပ်သည်။

```sql
-- Outbox table ဖန်တီးသည်
CREATE TABLE outbox_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,    -- entity အမျိုးအစား
    aggregate_id    VARCHAR(100) NOT NULL,    -- entity ID
    event_type      VARCHAR(100) NOT NULL,    -- event အမျိုးအစား
    payload         JSONB NOT NULL,           -- event data
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    published_at    TIMESTAMPTZ,             -- publish ဖြစ်သောချိန် (null = pending)
    published       BOOLEAN DEFAULT FALSE    -- publish ဖြစ်ပြီးလား
);
```

```python
# ✅ CORRECT — Transactional Outbox pattern
class OrderService:

    def place_order(self, command):
        with self.db.transaction():   # SINGLE TRANSACTION
            # Step 1: Business data save
            order = Order.create(command)
            self.db.save(order)         # orders table ထဲ save

            # Step 2: Outbox table ထဲ event save (same transaction!)
            self.db.execute("""
                INSERT INTO outbox_events
                (aggregate_type, aggregate_id, event_type, payload)
                VALUES (%s, %s, %s, %s)
            """, (
                'Order',
                str(order.id),
                'OrderPlaced',
                json.dumps({                        # event payload
                    'orderId': str(order.id),
                    'customerId': str(order.customer_id),
                    'totalAmount': float(order.total),
                    'items': order.items_as_list()
                })
            ))
            # Transaction commit — DB fail ဖြစ်ရင် နှစ်ခုလုံး rollback
```

### Outbox Pattern Flow

```
┌────────────────────────────────────────────────────────┐
│                  Single DB Transaction                 │
│  ┌──────────────┐         ┌────────────────────────┐  │
│  │  orders      │         │  outbox_events         │  │
│  │ (order saved)│         │ (event saved, pending) │  │
│  └──────────────┘         └────────────────────────┘  │
└────────────────────────────────────────────────────────┘
                                     │
                          Outbox Relay Process
                                     │
                                     ↓
                            [Message Broker]
                         (Kafka / RabbitMQ / SQS)
```

---

## ၁၃.၃ Polling Publisher

**Polling Publisher** သည် outbox table ကို periodic interval ဖြင့် poll လုပ်ကာ publish မဖြစ်ရသေးသော events များကို broker ထဲ publish ဆောင်ရွက်သည်။

```python
# Polling Publisher implementation
class OutboxPollingPublisher:

    def __init__(self, db, message_broker):
        self.db = db
        self.message_broker = message_broker

    def run(self):
        """Background thread တွင် run ဖြစ်သည်"""
        while True:
            self._process_pending_events()
            time.sleep(0.5)   # 500ms interval ဖြင့် poll

    def _process_pending_events(self):
        with self.db.transaction():
            # publish မဖြစ်ရသေးသော events ကို lock ဖြင့် ယူသည်
            # FOR UPDATE SKIP LOCKED = concurrent publishers safe
            pending_events = self.db.query("""
                SELECT * FROM outbox_events
                WHERE published = FALSE
                ORDER BY created_at ASC
                LIMIT 100
                FOR UPDATE SKIP LOCKED   -- concurrent race condition ကာကွယ်
            """)

            for event in pending_events:
                try:
                    # Broker ထဲ publish
                    self.message_broker.publish(
                        topic=f"{event.aggregate_type.lower()}.{event.event_type.lower()}",
                        key=event.aggregate_id,
                        value=event.payload,
                        headers={'eventId': str(event.id)}
                    )
                    # Published ဖြစ်ကြောင်း mark လုပ်သည်
                    self.db.execute("""
                        UPDATE outbox_events
                        SET published = TRUE,
                            published_at = NOW()
                        WHERE id = %s
                    """, (event.id,))
                except Exception as e:
                    print(f"Failed to publish {event.id}: {e}")
                    # retry လုပ်မည် — next poll cycle တွင်
                    break

    def cleanup_old_events(self):
        """7 ရက်ကြာသော published events ဖျက်ပစ်သည်"""
        self.db.execute("""
            DELETE FROM outbox_events
            WHERE published = TRUE
              AND published_at < NOW() - INTERVAL '7 days'
        """)
```

### Polling Publisher ၏ ကန့်သတ်ချက်

```
ကောင်းသောချက်:
✅ ရိုးရှင်း — SQL polling သာ
✅ Application code မှ implement ဖြစ်
✅ Any database ဖြင့် အလုပ်ဖြစ်

ဆုတ်ကျိုးချက်:
⚠ Latency — polling interval ကြောင့် delay ဖြစ်နိုင်
⚠ DB load — polling ကြောင့် extra queries
⚠ Scale ရန် ခက် — multiple instances ၏ coordination
```

---

## ၁၃.၄ Transaction Log Tailing with Debezium (CDC)

**Change Data Capture (CDC)** သည် database ၏ **transaction log** (WAL — Write-Ahead Log) ကို ဖတ်ပြီး change events ကို stream ထုတ်ပေးသော technique ဖြစ်သည်။ **Debezium** သည် open-source CDC tool ဖြစ်ပြီး Kafka Connect ကို အသုံးပြုသည်။

### CDC Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   PostgreSQL Server                     │
│  ┌──────────┐       ┌─────────────────────────────┐    │
│  │  orders  │──────→│  WAL (Write-Ahead Log)      │    │
│  │  table   │ write │  [INSERT order row]         │    │
│  │          │       │  [UPDATE order status]      │    │
│  └──────────┘       └──────────────┬──────────────┘    │
└─────────────────────────────────────┼───────────────────┘
                                      │ CDC reads log
                                      ↓
                             ┌──────────────────┐
                             │  Debezium        │
                             │  Connector       │
                             └────────┬─────────┘
                                      │ publishes changes
                                      ↓
                             ┌──────────────────┐
                             │   Kafka Topic    │
                             │ postgres.public  │
                             │  .outbox_events  │
                             └──────────────────┘
```

### Debezium Configuration

```json
// Debezium PostgreSQL Connector configuration
{
  "name": "orders-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",           // DB host
    "database.port": "5432",
    "database.user": "debezium",               // CDC user
    "database.password": "secret",
    "database.dbname": "orders_db",            // database အမည်
    "table.include.list": "public.outbox_events",  // outbox table မှ CDC
    "plugin.name": "pgoutput",                 // WAL plugin
    "transforms": "outbox",
    "transforms.outbox.type":
      "io.debezium.transforms.outbox.EventRouter",   // Outbox transformer
    "transforms.outbox.route.by.field": "aggregate_type",
    "transforms.outbox.table.fields.additional.placement":
      "event_type:header:eventType"            // header ထဲ eventType
  }
}
```

### CDC vs Polling နှိုင်းယှဉ်

```
┌──────────────────────┬──────────────────┬──────────────────────┐
│ Aspect               │ Polling Publisher│ CDC (Debezium)       │
├──────────────────────┼──────────────────┼──────────────────────┤
│ Latency              │ 500ms - 5sec     │ Near real-time (<1s) │
│ DB Load              │ Extra queries    │ WAL ဖတ်ရုံသာ         │
│ Implementation       │ Simple           │ Complex (infra)      │
│ Ordering guarantee   │ created_at sort  │ WAL order            │
│ All DB types         │ ✅ Yes           │ ⚠ Supported DBs only │
│ Schema dependency    │ outbox table     │ outbox table         │
└──────────────────────┴──────────────────┴──────────────────────┘
```

---

## ၁၃.၅ Inbox Pattern for Idempotent Consumers

**Inbox Pattern** သည် consumer side တွင် duplicate message ကိုကာကွယ်သော complementary pattern ဖြစ်သည်။ Received event ID ကို **inbox table** တွင် သိမ်းဆည်းကာ duplicate detect ဆောင်ရွက်သည်။

```sql
-- Inbox table
CREATE TABLE inbox_events (
    event_id        UUID PRIMARY KEY,          -- unique event ID
    event_type      VARCHAR(100) NOT NULL,
    processed_at    TIMESTAMPTZ DEFAULT NOW(),
    consumer_group  VARCHAR(100) NOT NULL      -- consumer service ID
);
```

```python
# Idempotent Consumer with Inbox Pattern
class OrderEventConsumer:

    def consume(self, message):
        event_id = message.headers.get('eventId')  # event ID ရယူ
        event_type = message.headers.get('eventType')

        with self.db.transaction():
            # duplicate စစ်ဆေး
            already_processed = self.db.query_one("""
                SELECT 1 FROM inbox_events
                WHERE event_id = %s
                  AND consumer_group = %s
            """, (event_id, 'inventory-service'))   # service ID ပါ

            if already_processed:
                print(f"Duplicate event {event_id}, skipping")
                return  # ထပ်ကြိမ် process မဖြစ်ရ

            # Business logic ဆောင်ရွက်
            self._handle_event(event_type, message.value)

            # Inbox ထဲ record လုပ်သည် (same transaction)
            self.db.execute("""
                INSERT INTO inbox_events
                (event_id, event_type, consumer_group)
                VALUES (%s, %s, %s)
            """, (event_id, event_type, 'inventory-service'))
            # Transaction commit — business logic နှင့် inbox တွဲသည်

    def _handle_event(self, event_type, data):
        """Actual business logic"""
        if event_type == 'OrderPlaced':
            self.inventory.reserve(data['items'])
        elif event_type == 'OrderCancelled':
            self.inventory.release(data['items'])
```

### Outbox + Inbox Combined Flow

```
Producer Side (Outbox):                Consumer Side (Inbox):
────────────────────────               ─────────────────────────
DB Transaction {                       DB Transaction {
  save(order)                            check inbox (no dup)
  insert(outbox_event)    →Event→        handle business logic
}                                        insert(inbox_event)
   ↓                                   }
CDC / Polling publishes
to Kafka
```

---

## ၁၃.၆ Schema Registry & Avro/Protobuf for Event Contracts

Event schema ကို service များကြား **contract** အဖြစ် formal ဖြင့် manage ရန် **Schema Registry** ကို အသုံးပြုသည်။

### Schema Registry Architecture

```
Producer                  Schema Registry           Consumer
────────                  ───────────────           ────────
serialize(event) ──→  register/validate schema ──→  deserialize(event)
  (Avro/Protobuf)      GET schema_id=42             check schema_id=42
```

### Apache Avro Schema ဥပမာ

```json
// order-placed.avsc — Avro schema
{
  "type": "record",
  "name": "OrderPlaced",
  "namespace": "com.myshop.orders.events",
  "fields": [
    {
      "name": "eventId",
      "type": "string",
      "doc": "Unique event identifier"           // event ID
    },
    {
      "name": "orderId",
      "type": "string",
      "doc": "Order identifier"                  // အမှာစာ ID
    },
    {
      "name": "customerId",
      "type": "string",
      "doc": "Customer identifier"               // ဖောက်သည် ID
    },
    {
      "name": "totalAmount",
      "type": {
        "type": "bytes",
        "logicalType": "decimal",
        "precision": 10,
        "scale": 2
      },
      "doc": "Total order amount"                // ငွေပမာဏ (decimal)
    },
    {
      "name": "currency",
      "type": ["null", "string"],               // optional field
      "default": null,
      "doc": "Currency code (e.g. MMK, USD)"    // ငွေကြေးအမျိုးအစား
    },
    {
      "name": "occurredAt",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      },
      "doc": "When the event occurred"          // ဖြစ်ပေါ်သည့်ချိန်
    }
  ]
}
```

### Protobuf Schema ဥပမာ

```protobuf
// order_events.proto
syntax = "proto3";
package com.myshop.orders.events;

message OrderPlacedEvent {
    string event_id     = 1;   // event ID
    string order_id     = 2;   // အမှာစာ ID
    string customer_id  = 3;   // ဖောက်သည် ID
    int64  total_amount = 4;   // ငွေပမာဏ (cents)
    string currency     = 5;   // ငွေကြေးအမျိုးအစား
    int64  occurred_at  = 6;   // timestamp (epoch ms)
    repeated OrderItem items = 7; // ပစ္စည်းများ
}

message OrderItem {
    string product_id = 1;     // product ID
    int32  quantity   = 2;     // အရေအတွက်
    int64  unit_price = 3;     // တစ်ခုစီ ငွေ
}
```

### Avro vs Protobuf နှိုင်းယှဉ်

```
┌──────────────────────┬──────────────────┬──────────────────────┐
│ Aspect               │ Avro             │ Protobuf             │
├──────────────────────┼──────────────────┼──────────────────────┤
│ Schema Evolution     │ ✅ Built-in      │ ✅ Field numbers     │
│ Schema Registry      │ ✅ Confluent SR  │ ✅ (with plugins)    │
│ Binary size          │ Small            │ Smaller              │
│ Code generation      │ Optional         │ Required             │
│ Kafka ecosystem fit  │ ✅ Best          │ ✅ Good              │
│ gRPC compatible      │ ❌               │ ✅ Native            │
│ Human readable       │ JSON schemas     │ .proto files         │
└──────────────────────┴──────────────────┴──────────────────────┘
```

### Schema Compatibility Modes (Schema Evolution)

Schema Registry တွင် schema version အသစ်တစ်ခု register ပြုသောအခါ compatibility check ဆောင်ရွက်သည်။ Compatibility mode သည် producer နှင့် consumer ကို independent deploy ဖြစ်ခွင့်ပေးသော အရေးကြီးသော setting ဖြစ်သည်။

```
┌────────────────────┬──────────────────────────────────────────────────┐
│ Compatibility Mode │ ရှင်းလင်းချက်                                     │
├────────────────────┼──────────────────────────────────────────────────┤
│ BACKWARD           │ schema အသစ်သည် schema အဟောင်းဖြင့် ရေးထားသော   │
│                    │ data ကို ဖတ်နိုင်ရမည်။ field ဖျက်ခြင်း သို့မဟုတ်│
│                    │ default ပါသော field အသစ်ထည့်ခြင်း ခွင့်ပြုသည်။  │
│                    │ Consumer ကို အရင် deploy ပြုပါ။                  │
├────────────────────┼──────────────────────────────────────────────────┤
│ FORWARD            │ schema အဟောင်းသည် schema အသစ်ဖြင့် ရေးထားသော   │
│                    │ data ကို ဖတ်နိုင်ရမည်။ field အသစ်ထည့်ခြင်း     │
│                    │ သို့မဟုတ် default ပါသော field ဖျက်ခြင်း         │
│                    │ ခွင့်ပြုသည်။ Producer ကို အရင် deploy ပြုပါ။     │
├────────────────────┼──────────────────────────────────────────────────┤
│ FULL               │ BACKWARD + FORWARD နှစ်ခုလုံး ကိုက်ညီရမည်။      │
│                    │ default ပါသော field သာ ထည့်/ဖျက် ခွင့်ပြုသည်။  │
│                    │ အလုံခြုံဆုံး mode ဖြစ်သည်။                      │
├────────────────────┼──────────────────────────────────────────────────┤
│ NONE               │ compatibility check မပြု — production တွင်       │
│                    │ မသုံးသင့်ပါ။                                     │
└────────────────────┴──────────────────────────────────────────────────┘
```

```
Schema Evolution ဥပမာ (BACKWARD compatible):

Version 1:                          Version 2 (BACKWARD compatible):
{                                   {
  "fields": [                         "fields": [
    {"name": "orderId", ...},           {"name": "orderId", ...},
    {"name": "customerId", ...},        {"name": "customerId", ...},
    {"name": "totalAmount", ...}        {"name": "totalAmount", ...},
                                        {"name": "currency",        ← field အသစ်
                                         "type": ["null","string"], ← default null
                                         "default": null}             (BACKWARD OK)
  ]                                   ]
}                                   }

Consumer (schema v2) သည် Producer (schema v1) ၏ data ကို ဖတ်နိုင်သည်
→ currency field သည် null default ရှိသောကြောင့် v1 data တွင် မပါလျှင် null ဖြစ်မည်
```

```python
# Schema Registry compatibility mode သတ်မှတ်ခြင်း
import requests

# Subject level compatibility config
requests.put(
    'http://schema-registry:8081/config/order-events-value',
    json={'compatibility': 'BACKWARD'},  # BACKWARD compatibility enforce
    headers={'Content-Type': 'application/vnd.schemaregistry.v1+json'}
)

# Schema register ပြုလုပ်ခြင်း — compatibility check auto ဖြစ်မည်
response = requests.post(
    'http://schema-registry:8081/subjects/order-events-value/versions',
    json={'schema': json.dumps(new_avro_schema)},
    headers={'Content-Type': 'application/vnd.schemaregistry.v1+json'}
)

if response.status_code == 409:
    # Compatibility check failed — schema ပြင်ရမည်
    print("Schema is not backward compatible!")
else:
    schema_id = response.json()['id']
    print(f"Schema registered with id: {schema_id}")
```

```python
# Confluent Schema Registry ဖြင့် Avro serialize/deserialize
from confluent_kafka.avro import AvroProducer, AvroConsumer
from confluent_kafka import avro

# Schema ဖတ်သည်
schema_str = open('order-placed.avsc').read()
value_schema = avro.loads(schema_str)

# Avro Producer
producer = AvroProducer({
    'bootstrap.servers': 'kafka:9092',
    'schema.registry.url': 'http://schema-registry:8081'  # SR URL
}, default_value_schema=value_schema)

# Event publish — schema validation ဖြစ်မည်
producer.produce(
    topic='order-events',
    value={
        'eventId': str(uuid.uuid4()),
        'orderId': 'ord-123',
        'customerId': 'cust-001',
        'totalAmount': bytes([0, 0, 0, 0, 235, 80]),  # 60,000 cents
        'currency': 'MMK',
        'occurredAt': int(time.time() * 1000)
    }
)
producer.flush()
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **Dual-Write Problem** သည် database write နှင့် message publish ကို atomic မဖြစ်ခြင်းကြောင့် data inconsistency ဖြစ်ပေါ်စေပြီး **Transactional Outbox Pattern** ဖြင့် event ကို same DB transaction တွင် outbox table ထဲ save ခြင်းဖြင့် ဖြေရှင်းနိုင်သည်။
2. **Polling Publisher** သည် implement ရန် ရိုးရှင်းသော်လည်း latency ကုန်ကျနိုင်ပြီး DB load တိုးစေသည်။ `FOR UPDATE SKIP LOCKED` ဖြင့် concurrent safety ရယူနိုင်သည်။
3. **CDC (Debezium)** သည် database ၏ WAL (Write-Ahead Log) ကို ဖတ်၍ near real-time change events ကို Kafka ထဲ stream ထုတ်ပေးပြီး Polling Publisher ထက် latency နည်းကာ DB load နည်းသည်။
4. **Inbox Pattern** ကို consumer side တွင် ထည့်ကာ at-least-once delivery ကြောင့် ဖြစ်နိုင်သော duplicate processing ကို event_id dedup ဖြင့် ကာကွယ်ပါ။ Outbox + Inbox ပေါင်းစပ်ခြင်းသည် reliable messaging ၏ complete solution ဖြစ်သည်။
5. **Schema Registry** နှင့် **Avro/Protobuf** ကို event contract enforcement အတွက် သုံးပြီး **BACKWARD, FORWARD, FULL** compatibility modes ဖြင့် schema evolution ကို safely ဆောင်ရွက်ပါ — breaking changes ကို production deploy မတိုင်မှီ registry level တွင် ရှာဖွေနိုင်မည်။
6. Production-grade event-driven system တွင် **Outbox table + CDC (Debezium) + Inbox table + Schema Registry** တို့ကို တွဲဖက်အသုံးပြုခြင်းသည် reliable, idempotent, schema-safe messaging ၏ foundation ဖြစ်သည်။
