# အခန်း ၁၆: Database per Service Pattern

## နိဒါန်း

Microservices architecture ၏ အရေးကြီးဆုံး principles တစ်ခုမှာ "Database per Service" pattern ဖြစ်သည်။ Monolithic application တွင် services အားလုံး database တစ်ခုတည်းကို share လုပ်ကြသည်ကို တွေ့ရသည်။ ဤ approach သည် ကနဦးတွင် ရိုးရှင်းပုံပေါ်သော်လည်း service boundaries ဖောက်ဖျက်ကာ tight coupling ဖြစ်စေသောကြောင့် microservices ၏ မူဝါဒနှင့် ကွဲလွဲသည်။ ဤအခန်းတွင် ဘာကြောင့် database sharing သည် ပြဿနာဖြစ်သည်ကို နားလည်ပြီး polyglot persistence နှင့် data consistency patterns များကို လေ့လာမည်ဖြစ်သည်။

---

## ၁၆.၁ Shared Databases ဘာကြောင့် Microservices ကို ချိုးဖောက်သည်

Shared database သည် ပေါ်ပေါ်ကြည့်လျှင် practical ဖြစ်ပုံပေါ်သည်။ သို့သော် depth တွင် ကြည့်သောအခါ microservices ၏ core benefits များကို ဖျက်ဆီးနေကြောင်း တွေ့ရသည်။

### ပြဿနာများ

```
Shared Database ဖြင့် Monolith Structure:

  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │  User       │   │  Order      │   │  Payment    │
  │  Service    │   │  Service    │   │  Service    │
  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
         │                 │                  │
         └─────────────────┼──────────────────┘
                           │
                    ┌──────▼──────┐
                    │  Shared     │
                    │  Database   │  ← SINGLE POINT OF FAILURE
                    │  (MySQL)    │  ← TIGHT COUPLING
                    └─────────────┘

ပြဿနာများ:
1. Schema changes → services အားလုံး affected ဖြစ်သည်
2. Service တစ်ခု database ကို overload လုပ်ပါက services အားလုံး slow ဖြစ်သည်
3. Services ကို independently deploy/scale မလုပ်နိုင်
4. Technology choices ကို lock-in ဖြစ်သည်
5. Team autonomy မရှိ
```

### Database per Service Architecture

```
  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │  User       │   │  Order      │   │  Payment    │
  │  Service    │   │  Service    │   │  Service    │
  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
         │                 │                  │
  ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
  │ PostgreSQL  │   │  MongoDB    │   │   Redis +   │
  │  (Users)   │   │  (Orders)   │   │  Cassandra  │
  └─────────────┘   └─────────────┘   └─────────────┘

အားသာချက်များ:
1. Independent schema evolution
2. Polyglot persistence (data type အလိုက် DB ရွေးနိုင်)
3. Independent scaling
4. Fault isolation
5. Team autonomy
```

---

## ၁၆.၂ Polyglot Persistence

"Polyglot Persistence" ဆိုသည်မှာ service တစ်ခုချင်းစီ၏ data nature အပေါ်မူတည်၍ သင့်တော်သော database technology ကို ရွေးချယ်သုံးစွဲခြင်းဖြစ်သည်။

### Database Types နှင့် Use Cases

**PostgreSQL — Relational Data**

```sql
-- E-commerce Order Service တွင် relational data
CREATE TABLE orders (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL,
    status      VARCHAR(50) NOT NULL DEFAULT 'pending',
    total_amount DECIMAL(10, 2) NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE order_items (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id    UUID REFERENCES orders(id),
    product_id  UUID NOT NULL,
    quantity    INT NOT NULL,
    unit_price  DECIMAL(10, 2) NOT NULL
);
```

**MongoDB — Document Store (Flexible Schema)**

```javascript
// Product Catalog Service — flexible product attributes
const productSchema = {
    _id: ObjectId,
    name: "Apple iPhone 15",
    category: "smartphones",
    // attributes သည် product type အလိုက် ကွဲပြားသည်
    attributes: {
        color: "Midnight",
        storage: "256GB",
        network: "5G",
        camera: "48MP"
    },
    variants: [
        { sku: "IP15-256-BLK", price: 999, stock: 50 },
        { sku: "IP15-512-BLK", price: 1199, stock: 30 }
    ],
    createdAt: new Date()
};
```

**Redis — Cache နှင့် Session Store**

```javascript
const redis = require('ioredis');
const client = new redis();

// Session data သိမ်းသည်
async function saveSession(sessionId, userData) {
    // JSON serialize လုပ်ပြီး 24 hours TTL ဖြင့် သိမ်းသည်
    await client.setex(
        `session:${sessionId}`,
        86400,
        JSON.stringify(userData)
    );
}

// Rate limiting counter
async function checkRateLimit(userId, maxRequests = 100) {
    const key = `ratelimit:${userId}:${Math.floor(Date.now() / 60000)}`;
    const count = await client.incr(key);
    if (count === 1) {
        await client.expire(key, 60); // 1 minute TTL
    }
    return count <= maxRequests;
}
```

**DynamoDB — Key-Value at Scale**

```javascript
const AWS = require('aws-sdk');
const dynamoDB = new AWS.DynamoDB.DocumentClient();

// User activity events သိမ်းသည်
async function recordEvent(userId, event) {
    await dynamoDB.put({
        TableName: 'UserEvents',
        Item: {
            userId,                              // Partition key
            timestamp: Date.now().toString(),    // Sort key
            eventType: event.type,
            eventData: event.data,
            // TTL: 90 days ကြာမှ auto-delete ဖြစ်သည်
            ttl: Math.floor(Date.now() / 1000) + (90 * 86400)
        }
    }).promise();
}
```

**InfluxDB — Time Series Data**

```javascript
const Influx = require('influx');

const influx = new Influx.InfluxDB({
    host: 'localhost',
    database: 'metrics',
});

// Metrics Service တွင် time series data သိမ်းသည်
async function recordMetric(serviceName, metricName, value) {
    await influx.writePoints([{
        measurement: metricName,
        tags: { service: serviceName },
        fields: { value },
        timestamp: new Date()
    }]);
}
```

**Neo4j — Graph Database**

```javascript
const neo4j = require('neo4j-driver');
const driver = neo4j.driver('bolt://localhost:7687');

// Recommendation Service တွင် user relationships
async function findRecommendations(userId) {
    const session = driver.session();
    try {
        // user ၏ friends ဝယ်ထားသော products ကို recommend လုပ်သည်
        const result = await session.run(`
            MATCH (u:User {id: $userId})-[:FRIEND]->(friend:User)
            MATCH (friend)-[:PURCHASED]->(product:Product)
            WHERE NOT (u)-[:PURCHASED]->(product)
            RETURN product.name AS recommendation, COUNT(*) AS score
            ORDER BY score DESC LIMIT 10
        `, { userId });

        return result.records.map(r => ({
            product: r.get('recommendation'),
            score: r.get('score').toInt()
        }));
    } finally {
        await session.close();
    }
}
```

**Elasticsearch — Full-Text Search**

```javascript
const { Client } = require('@elastic/elasticsearch');
const esClient = new Client({ node: 'http://localhost:9200' });

// Search Service — product search
async function searchProducts(query, filters) {
    const result = await esClient.search({
        index: 'products',
        body: {
            query: {
                bool: {
                    must: [
                        { match: { name: query } }
                    ],
                    filter: [
                        // price range filter
                        { range: { price: { gte: filters.minPrice, lte: filters.maxPrice } } },
                        { term: { category: filters.category } }
                    ]
                }
            },
            // highlight search term
            highlight: { fields: { name: {}, description: {} } }
        }
    });

    return result.hits.hits.map(hit => ({
        ...hit._source,
        score: hit._score,
        highlights: hit.highlight
    }));
}
```

**Cassandra — High Write Throughput**

```javascript
const cassandra = require('cassandra-driver');
const client = new cassandra.Client({ contactPoints: ['localhost'] });

// IoT Sensor Data — high write throughput
async function recordSensorData(deviceId, readings) {
    const query = `
        INSERT INTO iot_data.sensor_readings 
        (device_id, timestamp, temperature, humidity, pressure)
        VALUES (?, ?, ?, ?, ?)
    `;

    // Batch insert လုပ်သည် — Cassandra ၏ strength ဖြစ်သည်
    const batch = readings.map(r => ({
        query,
        params: [deviceId, r.timestamp, r.temperature, r.humidity, r.pressure]
    }));

    await client.batch(batch, { prepare: true });
}
```

---

## ၁၆.၃ Data Consistency Across Services

Services ကြား data consistency ထိန်းသိမ်းခြင်းသည် distributed system ၏ အကြီးမားဆုံး challenges တစ်ခုဖြစ်သည်။

### Saga Pattern

Saga pattern သည် distributed transactions ကို local transactions chain ဖြင့် implement လုပ်သည်။

```
Order Placement Saga:

┌──────────────────────────────────────────────────────────────┐
│                    Saga Orchestrator                         │
└──┬───────────────────────────────────────────────────────────┘
   │
   │  Step 1: Create Order
   ├──────────────────► Order Service ──► DB (pending status)
   │
   │  Step 2: Reserve Inventory
   ├──────────────────► Inventory Service ──► DB (reserved)
   │
   │  Step 3: Process Payment
   ├──────────────────► Payment Service ──► DB (charged)
   │
   │  Step 4: Confirm Order
   └──────────────────► Order Service ──► DB (confirmed)

Failure Handling (Compensating Transactions):
   Payment FAIL → Release Inventory → Cancel Order
```

```javascript
class OrderSaga {
    constructor(orderService, inventoryService, paymentService) {
        this.orderService = orderService;
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
    }

    async execute(orderData) {
        let orderId, reservationId;

        try {
            // Step 1: Order ဖန်တီးသည်
            orderId = await this.orderService.createOrder(orderData);

            // Step 2: Inventory reserve လုပ်သည်
            reservationId = await this.inventoryService.reserve(
                orderData.items
            );

            // Step 3: Payment process လုပ်သည်
            await this.paymentService.charge(
                orderData.userId,
                orderData.totalAmount
            );

            // Step 4: Order confirm လုပ်သည်
            await this.orderService.confirm(orderId);

            return { success: true, orderId };

        } catch (error) {
            // Compensating transactions: ပြောင်းပြန် undo လုပ်သည်
            console.error('Saga failed, compensating...', error);

            if (reservationId) {
                // inventory release လုပ်သည်
                await this.inventoryService.release(reservationId);
            }
            if (orderId) {
                // order cancel လုပ်သည်
                await this.orderService.cancel(orderId, error.message);
            }

            return { success: false, error: error.message };
        }
    }
}
```

### Event Sourcing with Eventual Consistency

```javascript
// Event store — all state changes ကို events ဖြင့် record လုပ်သည်
class EventStore {
    constructor(db) {
        this.db = db;
    }

    async appendEvent(aggregateId, eventType, eventData, expectedVersion) {
        // optimistic locking ဖြင့် concurrent writes ကာကွယ်သည်
        await this.db.query(`
            INSERT INTO events (aggregate_id, event_type, event_data, version)
            SELECT $1, $2, $3, COALESCE(MAX(version), 0) + 1
            FROM events
            WHERE aggregate_id = $1
            HAVING COALESCE(MAX(version), 0) = $4
        `, [aggregateId, eventType, JSON.stringify(eventData), expectedVersion]);
    }

    async getEvents(aggregateId, fromVersion = 0) {
        const result = await this.db.query(`
            SELECT * FROM events
            WHERE aggregate_id = $1 AND version > $2
            ORDER BY version ASC
        `, [aggregateId, fromVersion]);
        return result.rows;
    }
}
```

---

## ၁၆.၄ API Composition Pattern

Services တစ်ခုချင်းစီ၏ data ကို aggregate လုပ်ရန် API Composition ကို အသုံးပြုသည်။

```javascript
// API Gateway တွင် data composition လုပ်သည်
class OrderDetailComposer {
    constructor(orderService, userService, productService) {
        this.orderService = orderService;
        this.userService = userService;
        this.productService = productService;
    }

    async getOrderDetail(orderId) {
        // Order data ကို ရယူသည်
        const order = await this.orderService.getOrder(orderId);

        // Parallel requests ဖြင့် performance မြှင့်သည်
        const [user, products] = await Promise.all([
            // User info ရယူသည်
            this.userService.getUser(order.userId),
            // Product details ရယူသည်
            Promise.all(
                order.items.map(item =>
                    this.productService.getProduct(item.productId)
                )
            )
        ]);

        // Data ကို compose လုပ်ပြန်ပို့သည်
        return {
            orderId: order.id,
            status: order.status,
            customer: {
                id: user.id,
                name: user.name,
                email: user.email
            },
            items: order.items.map((item, index) => ({
                product: products[index].name,
                quantity: item.quantity,
                price: item.unitPrice
            })),
            total: order.totalAmount
        };
    }
}
```

### CQRS (Command Query Responsibility Segregation)

```
Write Model (Commands):
   ┌─────────────┐    ┌──────────────┐    ┌────────────────┐
   │  API        │───►│  Command     │───►│  Write DB      │
   │  Request    │    │  Handler     │    │  (PostgreSQL)  │
   └─────────────┘    └──────┬───────┘    └────────────────┘
                              │ publish events
                              ▼
                       ┌──────────────┐
                       │  Event Bus   │
                       └──────┬───────┘
                              │ sync
                              ▼
Read Model (Queries):         │
   ┌─────────────┐    ┌──────▼───────┐    ┌────────────────┐
   │  API        │───►│  Query       │───►│  Read DB       │
   │  Request    │    │  Handler     │    │  (Elasticsearch│
   └─────────────┘    └──────────────┘    │   or Redis)    │
                                          └────────────────┘
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

- **Database per Service:** Service တစ်ခုချင်းစီ ၎င်း၏ database ကို သီးသန့်ပိုင်ဆိုင်ရမည်ဖြစ်ပြီး other services မှ direct access ကို တားမြစ်ရသည် — API ဖြင့်သာ data ရယူနိုင်ရမည်
- **Polyglot Persistence:** Relational data တွင် PostgreSQL၊ flexible documents တွင် MongoDB၊ time-series data တွင် InfluxDB၊ graph data တွင် Neo4j ကဲ့သို့ data nature အပေါ်မူတည်၍ optimal database ကို ရွေးချယ်သင့်သည်
- **Eventual Consistency:** Distributed system တွင် cross-service strong consistency ကို အမြဲမထိန်းနိုင်သဖြင့် eventual consistency ကို accept ပြီး design လုပ်ရသောနေရာများ များသည် — Saga pattern ဖြင့် distributed transactions ကို handle လုပ်နိုင်သည်
- **API Composition:** Services ကြား data join လုပ်ရာတွင် API gateway level တွင် parallel requests ပေးပြီး data aggregate လုပ်သည် — N+1 problem ကို ရှောင်ရှားရန် data loader pattern သုံးနိုင်သည်
- **CQRS:** Write operations နှင့် Read operations ကို သီးခြား data models ဖြင့် handle လုပ်ခြင်းသည် performance ကောင်းမွန်ပြီး scale လုပ်ရ ပိုလွယ်ကူသည်
