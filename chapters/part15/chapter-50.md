# အခန်း ၅၀: Event-Driven Patterns at Scale

## နိဒါန်း

Event-driven architecture (EDA) သည် microservices world ၏ powerful pattern တစ်ခုဖြစ်ပြီး services များကြားတွင် loose coupling, scalability, resilience တို့ကို ဖြစ်နိုင်စေသည်။ သို့ရာတွင် scale ကြီးလာသည်နှင့်အမျှ challenges များသည် လည်းပိုမိုရှုပ်ထွေးလာသည်: event ordering, duplicate processing, schema evolution, multi-region replication — ဤတို့ကို careful design ဖြင့် ကိုင်တွယ်ရမည်ဖြစ်သည်။

ဤအခန်းတွင် large-scale event-driven systems တွင် apply ပြုသော advanced patterns အမျိုးမျိုးကို သွားမည်ဖြစ်ပြီး production experience မှ ပေါ်ထွက်လာသော practical wisdom များကို share ပြုမည်ဖြစ်သည်။

---

## ၅၀.၁ Competing Consumers & Partitioned Consumers

### Competing Consumers Pattern

Multiple consumer instances တို့သည် single queue မှ messages ကို ယှဉ်ပြိုင်ကာ consume ပြုသည်။

```
+----------+                    +------------------+
|          |  --> Consumer 1 -->|                  |
| Message  |  --> Consumer 2 -->|  Message Queue   |
| Queue    |  --> Consumer 3 -->|  (Kafka Topic)   |
|          |  --> Consumer 4 -->|                  |
+----------+                    +------------------+

Use case:
- Order processing (each order independent)
- Email sending (each email independent)
- Image resizing (each image independent)

Scaling: Add more consumers = more throughput
```

```python
class CompetingConsumerWorker:
    """
    Competing consumers - stateless workers
    Any worker can process any message
    """
    
    def __init__(self, kafka_consumer, processor):
        self.consumer = kafka_consumer
        self.processor = processor
    
    async def run(self):
        async for message in self.consumer:
            try:
                await self.processor.process(message.value)
                await self.consumer.commit()  # Manual commit after success
                
            except ProcessingError as e:
                # Retry or DLQ
                if message.retry_count < 3:
                    await self.requeue(message, delay=exponential_backoff(message.retry_count))
                else:
                    await self.dead_letter_queue.send(message)
```

### Partitioned Consumers Pattern

Kafka partitions ဖြင့် ordering guarantee + parallelism ကို တပြိုင်နက် ရရှိနိုင်သည်။

```
Topic: "user-events" (6 partitions)

Partition 0: User 1, User 7, User 13, ...  → Consumer A
Partition 1: User 2, User 8, User 14, ...  → Consumer B
Partition 2: User 3, User 9, User 15, ...  → Consumer C
Partition 3: User 4, User 10, User 16, ... → Consumer D
Partition 4: User 5, User 11, User 17, ... → Consumer E
Partition 5: User 6, User 12, User 18, ... → Consumer F

KEY INSIGHT: Same user's events → Same partition → Same consumer
→ Per-user ordering is guaranteed!
→ 6 consumers can process in parallel!
```

```python
class PartitionedConsumerManager:
    
    def assign_partition_key(self, user_id):
        """
        User ID ကို partition key အဖြစ် သုံး
        → Same user's events always go to same partition
        """
        return str(user_id)  # Kafka hash(key) % num_partitions
    
    async def publish_user_event(self, user_id, event_type, event_data):
        await self.kafka.send(
            topic="user-events",
            key=self.assign_partition_key(user_id),  # Ordering key
            value={
                "user_id": user_id,
                "event_type": event_type,
                "data": event_data,
                "timestamp": time.time()
            }
        )
```

### Consumer Group Rebalancing

```
Normal state:
Topic Partitions: P0 P1 P2 P3
Consumers:         C1 C2 C3 C4
Assignment:     C1=P0, C2=P1, C3=P2, C4=P3

Consumer C2 crashes:
Rebalancing triggered
New assignment: C1=P0,P1, C3=P2, C4=P3

// During rebalancing: Brief pause in processing
// Session timeout: 30 seconds (default)
// Rebalance complete: ~10-15 seconds

// Mitigation: 
// - Increase session.timeout.ms
// - Use incremental cooperative rebalancing (Kafka 2.4+)
```

---

## ၅၀.၂ Event Choreography vs Orchestration — Deep Trade-offs

### Choreography (Event-Driven)

```
No central coordinator - Services react to events directly

Order Service → publishes "order.created"
                    |
        +-----------+----------+----------+
        |           |          |          |
  Inventory     Payment    Warehouse  Notification
  Service       Service    Service    Service
  (reserves)    (charges)  (prepares) (notifies)
  
  → Each service listens to events and reacts

Pros:
+ Decentralized - no single coordinator
+ Services truly independent
+ Easy to add new consumers without changing publishers
+ Resilient (coordinator failure doesn't cascade)

Cons:
- Difficult to understand overall flow
- Business logic scattered across services
- Hard to track saga progress
- Testing complex (need all services)
- Cyclic dependencies possible
```

### Orchestration (Saga Orchestrator)

```
Central orchestrator controls the flow

+------------------+
|  Saga Orchestrator|
|  (Order Saga)    |
+------------------+
    |     |     |
    v     v     v
  Inv.  Pay.  Ware.
  Svc   Svc   Svc

Orchestrator:
1. "Reserve inventory" → Inventory Service
2. Inventory confirms → "Charge payment" → Payment Service
3. Payment confirms → "Create shipment" → Warehouse Service
4. Any failure → Run compensating transactions

Pros:
+ Single place to understand the flow
+ Explicit error handling
+ Easy to track saga state
+ Easier debugging

Cons:
- Orchestrator = central SPOF
- Orchestrator knows too much (coupling)
- Orchestrator can become a bottleneck
- More initial implementation work
```

### When to Use Which?

| Criteria | Choreography | Orchestration |
|----------|-------------|---------------|
| Process complexity | Simple, few steps | Complex, many steps |
| Error handling | Simple rollbacks | Complex compensations |
| Team autonomy | High (each team owns service) | Lower (shared orchestrator) |
| Debugging | Difficult | Easier (central state) |
| Business requirement | Loose coupling | Clear business flow |
| Example | Notification fanout | Order fulfillment saga |

---

## ၅၀.၃ Event Versioning Strategies

Production systems တွင် event schema ကို change မလုပ်ဘဲ မနေနိုင်ပါ။ Consumers ကို break မဖြစ်စေဘဲ schema evolve ပြုရသည်မှာ ကြီးမားသော challenge ဖြစ်သည်။

### Strategy 1: Additive Changes (Non-Breaking)

```json
// v1 Event
{
  "order_id": "ord_123",
  "user_id": "usr_456",
  "total": 50.00
}

// v2 Event (additive - new optional field added)
{
  "order_id": "ord_123",
  "user_id": "usr_456",
  "total": 50.00,
  "currency": "USD",    ← NEW optional field
  "promo_code": null    ← NEW optional field
}

// Old consumers: ignore unknown fields → still work ✓
// New consumers: use new fields ✓
// KEY RULE: Never remove/rename existing fields!
```

### Strategy 2: Event Versioning

```python
class EventRouter:
    """
    Event version routing - old consumers ကို protect
    """
    
    def publish_order_created(self, order):
        event = {
            "event_type": "order.created",
            "schema_version": "2.0",    ← Version included
            "event_id": str(uuid.uuid4()),
            "order_id": order.id,
            "user_id": order.user_id,
            "total": order.total,
            "currency": order.currency,   # v2 field
            "line_items": order.items     # v2 field
        }
        
        # Publish to versioned topics
        # Old consumers subscribe to v1 topic
        # New consumers subscribe to v2 topic
        self.kafka.publish("orders.created.v2", event)
        
        # Also publish downgraded v1 event for old consumers
        v1_event = self.downgrade_to_v1(event)
        self.kafka.publish("orders.created.v1", v1_event)
```

### Strategy 3: Schema Registry + Upcasting

```
Confluent Schema Registry Flow:

Producer → Schema Registry (register schema)
         ← Schema ID returned

Producer encodes: [magic byte][schema_id][avro_data]

Consumer → Schema Registry (fetch schema by ID)
         ← Schema returned
Consumer decodes avro_data with schema

// Backward compatible: New consumer can read old messages
// Forward compatible: Old consumer can read new messages
// Full compatible: Both ← IDEAL

// Upcasting: Old event → Newest version transform
class EventUpcaster:
    def upcast_order_v1_to_v2(self, v1_event):
        """v1 event ကို v2 format သို့ transform"""
        return {
            **v1_event,
            "currency": "USD",          # Default for v1 events
            "schema_version": "2.0"
        }
```

---

## ၅၀.၄ Temporal Decoupling & Replay for New Services

Event-driven systems ၏ powerful feature တစ်ခုမှာ new services ကို existing event history မှ bootstrap လုပ်နိုင်ခြင်းဖြစ်သည်။

### Event Replay Use Cases

```
Use Case 1: New service deployment
  - Analytics Service ကို new ထည့်လိုသည်
  - Historical orders (past 2 years) ကို analyze လုပ်လိုသည်
  - Solution: Kafka retention policy ဖြင့် replay ပြုလုပ်
  
  Kafka topic: "orders.created" (2-year retention)
  New Analytics Consumer: offset=0 မှ စဖတ် → process all historical events

Use Case 2: Bug fix and reprocess
  - Payment calculation bug ကို fix ပြုလိုသည်
  - Affected orders ကို reprocess လုပ်လိုသည်
  - Solution: Replay specific time range

Use Case 3: New feature backfill
  - Recommendation engine ကို အသစ် add လိုသည်
  - Historical user behavior ကို process လုပ်ရမည်
  - Solution: Historical event replay
```

### Replay Implementation

```python
class EventReplayService:
    
    async def replay_events(
        self,
        topic: str,
        consumer_group: str,
        from_timestamp: datetime,
        to_timestamp: datetime,
        processor
    ):
        """
        Specific time range ၏ events ကို replay
        """
        # Separate consumer group for replay (production မ impact မဖြစ်ရ)
        replay_consumer = KafkaConsumer(
            topic=topic,
            group_id=f"{consumer_group}_replay_{int(time.time())}",
            auto_offset_reset="earliest"
        )
        
        # Find offsets for time range
        partitions = replay_consumer.partitions_for_topic(topic)
        
        start_offsets = replay_consumer.offsets_for_times({
            TopicPartition(topic, p): int(from_timestamp.timestamp() * 1000)
            for p in partitions
        })
        
        # Seek to start offsets
        for tp, offset_and_timestamp in start_offsets.items():
            if offset_and_timestamp:
                replay_consumer.seek(tp, offset_and_timestamp.offset)
        
        processed_count = 0
        
        async for message in replay_consumer:
            if message.timestamp > int(to_timestamp.timestamp() * 1000):
                break  # Past end time
            
            await processor.process(message.value)
            processed_count += 1
            
            if processed_count % 10000 == 0:
                print(f"Replayed {processed_count} events...")
        
        print(f"Replay complete: {processed_count} events processed")
```

### Idempotent Event Processing (Critical for Replay)

```python
class IdempotentEventProcessor:
    """
    Event ကို multiple times process ပြုသောအခါ same result ဖြစ်ရမည်
    Replay scenarios အတွက် မဖြစ်မနေ implement ပြုရမည်
    """
    
    async def process(self, event):
        event_id = event["event_id"]
        
        # Deduplication check
        already_processed = await self.redis.get(f"processed:{event_id}")
        if already_processed:
            return  # Skip duplicate
        
        # Process event
        await self.handle_event(event)
        
        # Mark as processed (TTL = 7 days for replay safety)
        await self.redis.setex(f"processed:{event_id}", 604800, "1")
```

---

## ၅၀.၅ Multi-Region Event Streaming

Global services အတွက် events ကို multiple regions တွင် available ဖြစ်ရမည်ဖြစ်သည်ဆိုသည်မှာ Kafka replication ကို cross-region manage ပြုရမည်ဆိုသည်ဖြစ်သည်။

### Active-Active Multi-Region

```
US-East Kafka Cluster           EU-West Kafka Cluster
+-------------------+           +-------------------+
| Topic: orders     |  <------> | Topic: orders     |
| Partitions: 0-11  |  Mirr-   | Partitions: 0-11  |
| Local producers   |  orMaker  | Local producers   |
| Local consumers   |           | Local consumers   |
+-------------------+           +-------------------+

// Challenge: Event ordering across regions
// Solution: Global event ID + timestamp-based merging

// Each event includes:
{
  "event_id": "us-east-uuid-123",   ← Region-prefixed ID
  "global_seq": 100023,             ← Global sequence (Snowflake ID)
  "origin_region": "us-east-1",
  ...
}
```

### Kafka MirrorMaker 2 Configuration

```yaml
# MirrorMaker 2 config (cross-region replication)
connectors:
  - name: us-east-to-eu-west
    connector.class: org.apache.kafka.connect.mirror.MirrorSourceConnector
    source.cluster.alias: us-east
    target.cluster.alias: eu-west
    topics: "orders.*,payments.*,users.*"
    replication.factor: 3
    
    # Conflict resolution for active-active
    # us-east events get prefix "us-east." in eu-west topic
    replication.policy.class: DefaultReplicationPolicy
```

### Event Ordering in Multi-Region

```
Problem:
  US-East: Order created at T=100 (local time)
  EU-West: Order updated at T=99 (local time, but actually T=102 global)
  
  EU-West replicates "update" to US-East
  US-East sees: create(100) then update(99) → Wrong order!

Solution: Hybrid Logical Clocks (HLC)

HLC = physical_time + logical_increment

// ဤ approach ဖြင့် causality ကို ထိန်းသိမ်းနိုင်သည်
```

```python
class HybridLogicalClock:
    """
    Hybrid Logical Clock - distributed systems ၌ causality tracking
    """
    
    def __init__(self):
        self.l = 0  # Logical time (max seen)
        self.c = 0  # Counter
    
    def send_event(self):
        """New event ကို generate"""
        wall_clock = int(time.time() * 1000)  # Milliseconds
        l_new = max(self.l, wall_clock)
        
        if l_new == self.l:
            self.c += 1
        else:
            self.l = l_new
            self.c = 0
        
        return (self.l, self.c)  # HLC timestamp
    
    def receive_event(self, l_recv, c_recv):
        """Remote event receive - update local clock"""
        wall_clock = int(time.time() * 1000)
        l_new = max(self.l, l_recv, wall_clock)
        
        if l_new == self.l == l_recv:
            self.c = max(self.c, c_recv) + 1
        elif l_new == self.l:
            self.c += 1
        elif l_new == l_recv:
            self.c = c_recv + 1
        else:
            self.c = 0
        
        self.l = l_new
        return (self.l, self.c)
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **Competing vs Partitioned Consumers:** Stateless processing (email send, image resize) တွင် Competing Consumers သည် simpler ဖြစ်သည်။ Per-entity ordering (user events in order) လိုအပ်ပါက Partitioned Consumers ကို ရွေးချယ်ပြီး partition key ကို carefully design ပြုရမည်။

2. **Choreography vs Orchestration:** ကြီးသော business processes (order fulfillment, loan approval) တွင် Orchestration သည် comprehensibility နှင့် error handling အတွက် ကောင်းသော်လည်း simple fan-out patterns (notification send) တွင် Choreography လုံလောက်သည်။

3. **Additive-Only Schema Changes:** Existing fields ကို never remove/rename ပြုရ။ New optional fields ကို add ပြုနိုင်ပြီး backward compatibility ကို always maintain ပြုရမည်ဟူသော discipline ကို team တစ်ရပ်လုံးတွင် enforce ပြုရမည်။

4. **Event Replay Value:** Kafka ၏ high retention (days to years) သည် new service bootstrap, bug fix reprocessing, analytics backfill တို့ကို feasible ဖြစ်စေသည်။ Idempotent processing ကို mandatory ဖြစ်အောင် design ပြုရမည်။

5. **Multi-Region Complexity:** Cross-region event ordering သည် physically impossible ဖြစ်သောကြောင့် (light-speed limit) causality ကို Hybrid Logical Clocks ဖြင့် track ပြုရမည်။ MirrorMaker 2 သည် cross-region Kafka replication ၏ standard tool ဖြစ်သည်။
