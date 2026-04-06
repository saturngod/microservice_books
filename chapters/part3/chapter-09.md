# အခန်း ၉ — Messaging Infrastructure

## နိဒါန်း

Event-driven microservices ကို တည်ဆောက်ရာတွင် မှန်ကန်သော messaging infrastructure ရွေးချယ်မှုသည် system ၏ ကောင်းမွန်မှု၊ scalability နှင့် reliability တို့ကို တိုက်ရိုက် သက်ရောက်သည်။ **Message Broker** နှင့် **Event Streaming Platform** တို့သည် ပုံသဏ္ဍာန်ဆင်တူသော်လည်း သဘောတရားနှင့် အသုံးပြုပုံ ကွဲပြားသည်။ ဤအခန်းတွင် Apache Kafka, RabbitMQ, AWS SQS/SNS/EventBridge တို့ကို အသေးစိတ် လေ့လာမည်။ ထို့ပြင် Dead Letter Queue နှင့် Backpressure ကဲ့သို့ production-critical concept များကိုလည်း ဆွေးနွေးမည်။

---

## ၉.၁ Message Brokers vs Event Streaming Platforms

### Message Broker

**Message Broker** (ဥပမာ — RabbitMQ, AWS SQS) သည် message များကို queue တွင် သိမ်းဆည်း၍ consumer ထံ deliver လုပ်ပြီးနောက် delete ပြုလုပ်သည်။ message ကို consume ပြီးဆုံးသောအခါ ၎င်းသည် ပျောက်ကွယ်သွားသည်။

```
Producer ──→ [Queue/Exchange] ──→ Consumer A
                                  (message consumed → deleted)
```

### Event Streaming Platform

**Event Streaming Platform** (ဥပမာ — Apache Kafka, AWS Kinesis) သည် event log တစ်ခုကို ထိန်းသိမ်းသည်။ Consumer တစ်ဦးက read ပြီးနောက်လည်း event သည် ရှိနေဆဲဖြစ်သည်။ Consumer အများစုသည် မတူညီသော offset မှ ဖတ်နိုင်သည်။

```
Producer ──→ [Event Log (persisted)] ──→ Consumer Group A (offset: 50)
                                    └──→ Consumer Group B (offset: 37)
                                    └──→ Consumer Group C (offset: 50)
                (events ကို မဖျက် — retention period ထိ သိမ်းသည်)
```

### နှိုင်းယှဉ်ချက်

```
┌──────────────────┬─────────────────────┬────────────────────────┐
│ Feature          │ Message Broker      │ Event Streaming        │
├──────────────────┼─────────────────────┼────────────────────────┤
│ Retention        │ Consumed → Deleted  │ Configurable (days)    │
│ Replay           │ မဖြစ်နိုင်           │ ဖြစ်နိုင် (offset reset)│
│ Ordering         │ Queue level         │ Partition level        │
│ Throughput       │ Moderate            │ Very High              │
│ Use case         │ Task queue, RPC     │ Stream processing      │
└──────────────────┴─────────────────────┴────────────────────────┘
```

---

## ၉.၂ Apache Kafka Deep Dive

### Kafka Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Kafka Cluster                        │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│  │ Broker 1 │   │ Broker 2 │   │ Broker 3 │               │
│  │ Topic A  │   │ Topic A  │   │ Topic A  │               │
│  │ Part 0   │   │ Part 1   │   │ Part 2   │               │
│  └──────────┘   └──────────┘   └──────────┘               │
└─────────────────────────────────────────────────────────────┘
    ↑ Producer writes          ↓ Consumer Group reads
```

### Topics & Partitions

**Topic** သည် event ၏ category တစ်ခုဖြစ်သည်။ Topic တစ်ခုကို **Partition** များစွာ ခွဲ၍ parallel processing ဖြစ်နိုင်သည်။

```python
# Kafka Producer ဥပမာ (Python)
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda k: k.encode('utf-8')
)

# event ပို့ပါ — key ဖြင့် partition သတ်မှတ်သည်
producer.send(
    topic='order-events',              # topic အမည်
    key='cust-001',                    # customer ID ကို key အဖြစ် သုံးသည်
    value={
        'eventType': 'OrderPlaced',    # event အမျိုးအစား
        'orderId': 'ord-123',          # အမှာစာ ID
        'amount': 45000                # ငွေပမာဏ
    }
)
producer.flush()  # buffer ကို flush လုပ်သည်
```

### Offsets & Consumer Groups

**Offset** သည် partition အတွင်း message ၏ position (ကိန်းဂဏန်း) ဖြစ်သည်။ **Consumer Group** တွင် consumer များစွာပါဝင်ပြီး partition တစ်ခုကို consumer တစ်ဦးသာ assign ဖြစ်သည်။

```
Topic: orders (3 partitions)
Partition 0: [msg0, msg1, msg2, msg3, msg4]
                                   ↑ Consumer A reads up to offset 4

Partition 1: [msg0, msg1, msg2]
                         ↑ Consumer B reads up to offset 2

Partition 2: [msg0, msg1, msg2, msg3]
                              ↑ Consumer C reads up to offset 3
```

```python
# Kafka Consumer ဥပမာ
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'order-events',                          # subscribe လုပ်မည့် topic
    bootstrap_servers=['kafka:9092'],
    group_id='inventory-service-group',       # consumer group ID
    auto_offset_reset='earliest',             # အစကနေ ဖတ်မည်
    enable_auto_commit=False,                 # manual commit သုံးမည်
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    try:
        event = message.value               # event data ယူသည်
        process_event(event)                # business logic လုပ်သည်
        consumer.commit()                   # offset ကို manual commit
    except Exception as e:
        print(f"Processing failed: {e}")    # error ကို log တင်သည်
        # dead letter queue ထဲ ပို့မည်
```

### Retention & Log Compaction

**Retention** — Kafka သည် event များကို configurable period (ဥပမာ 7 ရက်) ထိ သိမ်းဆည်းသည်။

**Log Compaction** — key ကို base ထားပြီး latest value သာ သိမ်းသည်။ Database-like behavior ဖြစ်စေသည်။

```
# Log Compaction ဥပမာ
Before compaction:
[key=user1: v1] [key=user2: v1] [key=user1: v2] [key=user1: v3]

After compaction:
[key=user2: v1] [key=user1: v3]   ← latest value သာ ကျန်သည်
```

### Kafka Streams

**Kafka Streams** သည် Kafka ကို library အဖြစ် သုံး၍ stream processing ပြုလုပ်သည်။

```java
// Kafka Streams ဥပမာ (Java)
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders =
    builder.stream("order-events");  // input topic

// filter → map → output topic
orders
    .filter((key, order) ->
        order.getAmount() > 100000)      // ငွေပမာဏ 100,000 ကျော်သာ
    .mapValues(order ->
        new LargeOrder(order))           // transform လုပ်သည်
    .to("large-order-events");           // output topic ထဲ ရေးသည်
```

### Kafka Connect

**Kafka Connect** သည် external system (database, file system) နှင့် Kafka ကို ချိတ်ဆက်သည်။

```
[MySQL DB] ──→ [Source Connector] ──→ [Kafka Topic] ──→ [Sink Connector] ──→ [Elasticsearch]
```

---

## ၉.၃ RabbitMQ

RabbitMQ သည် AMQP protocol ကို အသုံးပြုသော message broker ဖြစ်သည်။ **Exchange**, **Queue**, **Binding** တို့သည် ၎င်း၏ အဓိက component များဖြစ်သည်။

### Exchange Types

```
Direct Exchange — routing key exact match
┌──────────────┐     routing_key="order"    ┌──────────────┐
│   Producer   │ ──────────────────────────→│  order_queue │
└──────────────┘                            └──────────────┘

Topic Exchange — wildcard routing
┌──────────────┐   routing_key="order.*"   ┌──────────────────┐
│   Producer   │ ─────────────────────────→│ order.created Q  │
└──────────────┘   routing_key="order.#"   └──────────────────┘

Fanout Exchange — broadcast (routing key မသုံး)
┌──────────────┐                           ┌──────────────┐
│   Producer   │ ──────────────────────→   │   Queue A    │
└──────────────┘        broadcast          ├──────────────┤
                                           │   Queue B    │
                                           ├──────────────┤
                                           │   Queue C    │
                                           └──────────────┘
```

```python
# RabbitMQ Producer ဥပမာ (Python pika)
import pika
import json

connection = pika.BlockingConnection(
    pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()

# exchange ကြေညာ
channel.exchange_declare(
    exchange='orders',
    exchange_type='topic',       # topic exchange သုံးသည်
    durable=True)                # restart ဖြစ်လည်း ရှိနေမည်

# message ပို့
channel.basic_publish(
    exchange='orders',
    routing_key='order.placed',  # routing key
    body=json.dumps({
        'orderId': 'ord-123',
        'customerId': 'cust-001'
    }),
    properties=pika.BasicProperties(
        delivery_mode=2,         # message persistent ဖြစ်သည်
    )
)
connection.close()
```

---

## ၉.၄ AWS SQS / SNS / EventBridge

### SQS (Simple Queue Service)

**SQS** သည် managed message queue service ဖြစ်သည်။ **Standard Queue** (at-least-once, unordered) နှင့် **FIFO Queue** (exactly-once, ordered) ဟူ၍ နှစ်မျိုးရှိသည်။

```
Producer ──→ [SQS Queue] ──→ Consumer (Lambda / EC2 / ECS)
                              (visibility timeout ကြာသောအခါ
                               process မပြီးဆုံးလျှင် ပြန်ပေါ်သည်)
```

### SNS (Simple Notification Service)

**SNS** သည် pub/sub service ဖြစ်သည်။ Topic တစ်ခုသို့ publish လုပ်သောအခါ subscriber များသို့ fan-out ဖြင့် deliver ဖြစ်သည်။

### EventBridge

**EventBridge** သည် event routing service ဖြစ်ပြီး rule-based filtering ဖြင့် target service ဆီ route ဖြစ်သည်။

```
[Order Service] ──→ [EventBridge Bus] ──→ Rule 1: source=orders → Lambda
                                     └──→ Rule 2: detail-type=OrderPlaced → SQS
                                     └──→ Rule 3: all events → S3 Archive
```

---

## ၉.၅ Kafka vs RabbitMQ vs SQS — Decision Guide

```
┌──────────────────────────┬──────────┬───────────┬──────────────┐
│ Scenario                 │ Kafka    │ RabbitMQ  │ SQS/SNS      │
├──────────────────────────┼──────────┼───────────┼──────────────┤
│ High-throughput events   │ ✅ Best  │ ⚠ Okay   │ ⚠ Okay      │
│ Event replay needed      │ ✅ Best  │ ❌ No     │ ❌ No        │
│ Complex routing          │ ⚠ Basic │ ✅ Best   │ ⚠ EventBridge│
│ Simple task queue        │ ⚠ Over  │ ✅ Good   │ ✅ Best      │
│ AWS managed infra        │ ⚠ MSK   │ ⚠ AmazonMQ│ ✅ Native    │
│ Stream processing        │ ✅ Best  │ ❌ No     │ ❌ No        │
│ Ordered delivery         │ Partition│ Queue     │ FIFO Queue   │
└──────────────────────────┴──────────┴───────────┴──────────────┘
```

---

## ၉.၆ Dead Letter Queues & Poison Messages

**Dead Letter Queue (DLQ)** သည် process မဖြစ်နိုင်သော (poison) message များကို သီးသန့် သိမ်းဆည်းသော queue ဖြစ်သည်။

```
Normal Flow:
Producer ──→ [Main Queue] ──→ Consumer
                               (retry 3 times → fail)
                                    │
                                    ↓ max retry ကျော်လျှင်
                              [Dead Letter Queue]
                                    │
                              Alert / Manual inspection
```

```python
# SQS DLQ configuration ဥပမာ (boto3)
import boto3

sqs = boto3.client('sqs')

# DLQ ဖန်တီးသည်
dlq_response = sqs.create_queue(
    QueueName='orders-dlq',             # dead letter queue အမည်
    Attributes={'MessageRetentionPeriod': '1209600'}  # 14 ရက် သိမ်း
)

# Main queue ၏ redrive policy သတ်မှတ်သည်
sqs.set_queue_attributes(
    QueueUrl='https://sqs.../orders-queue',
    Attributes={
        'RedrivePolicy': json.dumps({
            'maxReceiveCount': '3',          # ၃ ကြိမ် retry ပြီးမှ DLQ
            'deadLetterTargetArn': dlq_arn   # DLQ ARN
        })
    }
)
```

---

## ၉.၇ Backpressure & Flow Control

**Backpressure** သည် downstream system မ handle နိုင်သောအခါ upstream ကို နှေးကွေးစေသော mechanism ဖြစ်သည်။

```
Fast Producer ──→ [Buffer/Queue] ──→ Slow Consumer
                 ↑ full!
         Producer throttled (backpressure applied)
```

```python
# Consumer ဘက်တွင် prefetch limit သတ်မှတ်ခြင်း (RabbitMQ)
channel.basic_qos(
    prefetch_count=10  # တစ်ချိန်တည်း unacked message ၁၀ ခုသာ
)

# Kafka တွင် max.poll.records သတ်မှတ်ခြင်း
consumer_config = {
    'max.poll.records': 50,           # တစ်ကြိမ်တွင် record ၅၀ ခုသာ
    'max.poll.interval.ms': 300000,   # processing timeout 5 မိနစ်
}
```

---

## အဓိကသင်ခန်းစာများ

- **Message Broker** သည် message consumed ပြီးနောက် delete ဖြစ်ပြီး **Event Streaming Platform** သည် replay ဖြစ်နိုင်သည်။
- Kafka တွင် **Partition** key ဖြင့် ordering ကို သေချာပြီး **Consumer Group** ဖြင့် parallel processing ဖြစ်နိုင်သည်။
- **Log Compaction** သည် key-based latest state ကို သိမ်းဆည်းသဖြင့် state reconstruction ဖြစ်နိုင်သည်။
- RabbitMQ ၏ **Exchange type** (direct, topic, fanout) သည် routing flexibility ကို ပေးသည်။
- **Dead Letter Queue** သည် poison message များကို isolate ပြီး system တစ်ခုလုံး ရပ်တန့်မသွားစေရန် ကာကွယ်သည်။
- **Backpressure** mechanism ဖြင့် consumer ၏ capacity ကို မကျော်လွန်အောင် flow ကို ထိန်းညှိသည်။
- Kafka သည် high-throughput event streaming နှင့် replay လိုသောအခါ၊ RabbitMQ သည် complex routing လိုသောအခါ၊ SQS သည် AWS-native simple queue လိုသောအခါ ရွေးချယ်ပါ။
