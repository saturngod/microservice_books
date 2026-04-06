# နောက်ဆက်တွဲ B: Message Broker နှိုင်းယှဉ်ဇယား — Kafka vs RabbitMQ vs SQS vs Pulsar

---

## B.1 အဓိက နှိုင်းယှဉ်ဇယား

| နှိုင်းယှဉ်မှုအတိုင်းအတာ | **Apache Kafka** | **RabbitMQ** | **Amazon SQS** | **Apache Pulsar** |
|------------------------|-----------------|-------------|---------------|------------------|
| **Protocol** | Custom binary (TCP) / Kafka Protocol | AMQP 0-9-1, MQTT, STOMP | AWS proprietary (HTTPS/REST) | Pulsar binary protocol |
| **Message Ordering** | Partition အတွင်း ordering အာမခံ | Queue တစ်ခုအတွင်း ordering (single consumer) | Standard: မအာမခံ; FIFO queue: ordering အာမခံ | Partition အတွင်း ordering အာမခံ |
| **Delivery Guarantees** | At-least-once (default); Exactly-once semantics ကို Kafka transactions + idempotent processing ဖြင့် တည်ဆောက်နိုင် | At-least-once; At-most-once (autoAck) | At-least-once (standard); FIFO တွင် ordered delivery + duplicate send suppression | At-least-once; Effectively-once (idempotent) |
| **Message Retention** | Time-based သို့မဟုတ် size-based (default 7 days); Log compaction | Consumer ack ပြီးသည့်နောက် ဖျက်သည် (default) | 1 minute – 14 days | Time-based / size-based; Tiered storage (S3) |
| **Max Throughput** | Very High (millions msg/sec per cluster) | Medium-High (~50K-200K msg/sec) | Medium (SQS has per-queue limits) | Very High (comparable to Kafka) |
| **Latency** | Low (~1-5ms end-to-end) | Very Low (sub-millisecond possible) | Medium (~10-50ms+) | Low (~5ms) |
| **Consumer Model** | Pull-based (consumer group) | Push + Pull (subscribe/get) | Pull-based (long polling) | Push + Pull (subscription types) |
| **Message Replay** | ဖြစ်နိုင်သည် (offset reset ဖြင့်) | မဖြစ်နိုင် (ack ပြီးလျှင် ပျောက်) | မဖြစ်နိုင် (ack ပြီးလျှင် ပျောက်) | ဖြစ်နိုင်သည် (cursor reset ဖြင့်) |
| **Topic / Queue Model** | Topics → Partitions | Exchanges → Queues (routing) | Queues တိုက်ရိုက် | Topics → Partitions (namespaced) |
| **Consumer Group Semantics** | Consumer group — partition-level load balancing | Competing consumers (queue-based) | Multiple consumers — each gets separate copy (SNS fanout) | Subscription types (shared, failover, key_shared) |
| **Persistence** | Disk-persistent (always) | Optional (persistent / transient) | Cloud-managed persistence | Disk + Tiered storage |
| **Schema Registry** | Confluent Schema Registry (add-on) | မပါ (external library) | မပါ | Built-in schema registry |
| **Multi-tenancy** | Limited (per-cluster config) | Virtual hosts | AWS account-level isolation | Built-in (namespaces + tenants) |
| **Geo-replication** | MirrorMaker 2 (manual) | Federation plugin | AWS cross-region SNS/SQS | Built-in geo-replication |
| **Managed Cloud Options** | Confluent Cloud, AWS MSK, Aiven | CloudAMQP, AWS MQ, Azure Service Bus | AWS SQS (native) | StreamNative Cloud, Aiven |
| **Open Source** | Yes (Apache License) | Yes (Mozilla Public License) | No (AWS proprietary) | Yes (Apache License) |
| **Learning Curve** | High | Medium | Low | High |
| **Operational Complexity** | High (ZooKeeper / KRaft) | Medium | Very Low (managed) | High |

---

## B.2 Use Case ကြည့် ဘာသုံးမည်ဆိုသော ဆုံးဖြတ်ဇယား

| Use Case | အကောင်းဆုံးရွေးချယ်မှု | အကြောင်းပြချက် |
|----------|----------------------|--------------|
| Event streaming / log aggregation | **Kafka** သို့မဟုတ် **Pulsar** | Retention + replay + high throughput |
| Simple task queue (decoupled workers) | **RabbitMQ** သို့မဟုတ် **SQS** | Easy setup, auto-delete after ack |
| Real-time analytics pipeline | **Kafka** | Log compaction, stream processing (Kafka Streams / Flink) |
| Microservices event-driven architecture | **Kafka** သို့မဟုတ် **Pulsar** | Event sourcing, audit log |
| AWS-native application | **SQS** + **SNS** | Zero ops, pay-per-use, tight AWS integration |
| Complex routing / message transformation | **RabbitMQ** | Exchange types (direct, fanout, topic, headers) |
| IoT device messaging | **Pulsar** သို့မဟုတ် **RabbitMQ** (MQTT) | MQTT support, geo-replication |
| Long-running job queue | **SQS** သို့မဟုတ် **RabbitMQ** | Visibility timeout, dead letter queue |
| Multi-tenant SaaS platform | **Pulsar** | Built-in tenant/namespace isolation |
| Exactly-once financial transactions | **Kafka** (transactions) | Idempotent producers + transactional API |

---

## B.3 Delivery Guarantee အသေးစိတ်

### At-most-once (မည်သည့်မျှ မပိုမဆီး)
```
Producer → Broker → Consumer
           ↓
        မ record မဒေတာ
        message ပျောက်နိုင်
        duplicate မဖြစ်
```
- ဥပမာ: Metrics, logs (ပျောက်သွားလည်း ဒဏ်ကြီး မခံရ)

### At-least-once (အနည်းဆုံး တစ်ကြိမ်)
```
Producer → Broker → Consumer
                    ↓
                 ack မရသေး → retry
                 duplicate ဖြစ်နိုင်
```
- ဥပမာ: Email notification (idempotent receiver လိုအပ်)

### Exactly-once (တိတိကျကျ တစ်ကြိမ်သာ)
```
Producer → Broker → Consumer
(idempotent)      (transactional)
```
- ဥပမာ: Payment processing, inventory update

---

## B.4 Kafka Consumer Group vs RabbitMQ Queue — ကွာခြားချက်

### Kafka Consumer Group
```
Topic: orders
├── Partition 0 ──→ Consumer A (Group 1)
├── Partition 1 ──→ Consumer B (Group 1)
└── Partition 2 ──→ Consumer C (Group 1)
                 ──→ Consumer X (Group 2)  ← အခြား group သည် အကုန် ကော်ပီ ရသည်
                 ──→ Consumer Y (Group 2)
```
- Group ခြင်း message share မလုပ် — parallel processing
- မတူညီသော group များသည် message အကုန် ရသည် (multicast)

### RabbitMQ Competing Consumers
```
Queue: orders
├── Message 1 ──→ Worker A
├── Message 2 ──→ Worker B
├── Message 3 ──→ Worker A
└── Message 4 ──→ Worker C
```
- Queue တစ်ခုမှ message တစ်ခုကို worker တစ်ယောက်သာ ရသည်
- Fanout လုပ်ရန် — exchange binding ဖြင့် queue များစွာ bind လုပ်ရ

---

## B.5 Dead Letter Queue (DLQ) — ယှဉ်ချ

| Feature | Kafka | RabbitMQ | SQS |
|---------|-------|---------|-----|
| **DLQ ရှိ/မရှိ** | Manual implement (error topic) | Built-in (`x-dead-letter-exchange`) | Built-in (DLQ attribute) |
| **DLQ trigger** | Application error handling | Max retry exceeded / TTL expired / queue full | Max receive count exceeded |
| **DLQ retention** | Kafka topic retention policy | Queue config | 1-14 days |
| **DLQ reprocessing** | Offset seek + replay | Message republish | Redrive policy |

---

## B.6 Cloud Managed Options အသေးစိတ်

| Provider | Service | Kafka Compatible? | Pricing Model |
|----------|---------|------------------|---------------|
| **AWS** | Amazon MSK | Yes (fully) | Per broker-hour + storage |
| **AWS** | Amazon SQS | No | Per million requests |
| **AWS** | Amazon MQ | RabbitMQ / ActiveMQ | Per broker-hour |
| **Confluent** | Confluent Cloud | Yes (plus extras) | CKU (Kafka Compute Units) |
| **GCP** | Pub/Sub | No (similar concept) | Per GB + per million operations |
| **Azure** | Event Hubs | Yes (Kafka API) | Throughput units |
| **Azure** | Service Bus | RabbitMQ-like | Per million operations |
| **StreamNative** | StreamNative Cloud | Pulsar | Per compute + storage |
| **Aiven** | Aiven for Kafka | Yes | Per node-hour |

---

## B.7 ရွေးချယ်မှုဆုံးဖြတ်ချက် Decision Tree

```
Message Broker လိုသလား?
│
├── AWS ကိုသာ သုံးမည်ဆိုလျှင်?
│   └── SQS (simple) + SNS (fanout)
│
├── Event replay / audit log လိုသလား?
│   ├── Multi-tenant platform ဖြစ်သလား?
│   │   └── Pulsar
│   └── Standard streaming pipeline?
│       └── Kafka
│
├── Complex routing / per-message TTL / priority queue လိုသလား?
│   └── RabbitMQ
│
└── Simple task queue, low ops overhead?
    ├── Cloud-agnostic? → RabbitMQ
    └── AWS-native?     → SQS
```

---

## B.8 Performance Benchmark ခန့်မှန်းချက်

> **သတိ:** ဤတန်ဖိုးများသည် hardware, configuration, payload size ပေါ်မူတည်၍ ကွဲပြားနိုင်သည်။

| Broker | Throughput (msg/sec) | P99 Latency | Notes |
|--------|---------------------|-------------|-------|
| Kafka | 1M – 10M+ | 2–10 ms | Batch size ကြီးလျှင် throughput မြင့်သည် |
| RabbitMQ | 50K – 200K | < 1 ms | Single node; queue mirror ဖြင့် ကျဆင်းသည် |
| SQS | ~3,000 msg/sec/queue | 10–50+ ms | Standard queue; FIFO queue ပိုနှေး |
| Pulsar | 1M – 5M+ | 3–10 ms | Tiered storage သုံးလျှင် latency မြင့်သည် |
