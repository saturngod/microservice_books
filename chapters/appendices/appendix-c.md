# နောက်ဆက်တွဲ C: Database ရွေးချယ်မှုလမ်းညွှန်

---

## C.1 SQL vs NoSQL vs NewSQL — အဓိကနှိုင်းယှဉ်

| နှိုင်းယှဉ်မှုအတိုင်းအတာ | **SQL (Relational)** | **NoSQL** | **NewSQL** |
|------------------------|--------------------|-----------| -----------|
| **Data Model** | Tables, rows, columns | Document, Key-Value, Column-family, Graph | Relational (SQL interface) |
| **Schema** | Strict schema (schema-on-write) | Flexible schema (schema-on-read) | Strict schema |
| **Query Language** | SQL (standardized) | Database-specific API / query lang | SQL (ANSI compatible) |
| **ACID** | အပြည့်အဝ ACID | BASE (Basically Available, Soft state, Eventual consistency) | အပြည့်အဝ ACID + distributed |
| **Horizontal Scaling** | ခက်ခဲသည် (sharding manual) | လွယ်ကူသည် (built-in) | မြင့်မားသည် (built-in sharding) |
| **Joins** | Strong support | Limited / application-level | Strong support |
| **Consistency** | Strong consistency | Eventual consistency (default) | Strong consistency (distributed) |
| **Best For** | Structured data, complex queries, transactions | High volume, flexible data, low latency | Global scale + ACID transactions |
| **ဥပမာများ** | PostgreSQL, MySQL, Oracle | MongoDB, Redis, Cassandra, DynamoDB | CockroachDB, Google Spanner, YugabyteDB |

---

## C.2 Database အမျိုးအစားများ — Use Case မူတည်၍ ရွေးချယ်မှုဇယား

| Use Case | အကြံပြု Database | အကြောင်းပြချက် |
|----------|----------------|--------------|
| OLTP (Online Transaction Processing) | **PostgreSQL**, MySQL | ACID, complex joins, mature ecosystem |
| E-commerce cart / session | **Redis** | Sub-millisecond latency, TTL support |
| User profiles (flexible schema) | **MongoDB** | Document model, nested data, horizontal scale |
| Time-series metrics / IoT data | **InfluxDB**, TimescaleDB | Time-series optimized, downsampling |
| Product catalog (full-text search) | **Elasticsearch** | Inverted index, fuzzy search, aggregations |
| Social graph / recommendation | **Neo4j**, Amazon Neptune | Graph traversal, relationship queries |
| Global distributed transactions | **CockroachDB**, Google Spanner | NewSQL, geo-distributed ACID |
| Write-heavy, high availability | **Cassandra**, DynamoDB | Wide-column, tunable consistency, multi-region |
| Caching layer | **Redis**, Memcached | In-memory, ultra-low latency |
| Event sourcing / audit log | **Kafka** + **PostgreSQL** | Append-only, replay capability |
| Leaderboard / ranking | **Redis** (Sorted Sets) | O(log n) rank queries |
| Geospatial queries | **PostgreSQL** (PostGIS), MongoDB | Geo index, nearest neighbor |
| Analytics / OLAP | **Snowflake**, BigQuery, ClickHouse | Columnar storage, vectorized execution |
| Multi-model (graph + document) | **ArangoDB** | Flexible model support |

---

## C.3 တစ်ခုချင်းစီ Database — ရှင်းလင်းချက်နှင့် အသုံးပြုပုံ

### PostgreSQL
- **အမျိုးအစား:** Relational (RDBMS)
- **CAP Position:** CA (Partition Tolerance ကို single-node ဖြင့် sacrifice)
- **အားသာချက်:** ACID, advanced SQL, JSON support (JSONB), PostGIS, full-text search
- **အားနည်းချက်:** Horizontal write scaling ခက်ခဲ (sharding: Citus သုံးရ)
- **သုံးသင့်သည့်အခါ:** Complex transactions, reporting, general-purpose OLTP
- **မသုံးသင့်သည့်အခါ:** Write-heavy global scale, schema-less flexible data

### MySQL
- **အမျိုးအစား:** Relational (RDBMS)
- **CAP Position:** CA
- **အားသာချက်:** Wide adoption, InnoDB engine, replication mature
- **အားနည်းချက်:** PostgreSQL နှင့် feature gap (window functions, JSON less mature)
- **သုံးသင့်သည့်အခါ:** Web applications, CMS (WordPress, Drupal), read-heavy workloads
- **မသုံးသင့်သည့်အခါ:** Complex analytical queries, PostGIS geospatial

### MongoDB
- **အမျိုးအစား:** Document (NoSQL)
- **CAP Position:** CP (default) / AP (tunable)
- **အားသာချက်:** Flexible schema, horizontal sharding built-in, aggregation pipeline
- **အားနည်းချက်:** No multi-document ACID (pre-v4.0); joins are application-level
- **သုံးသင့်သည့်အခါ:** Product catalog, user profiles, content management, real-time apps
- **မသုံးသင့်သည့်အခါ:** Financial transactions requiring strict ACID across collections

### Redis
- **အမျိုးအစား:** In-memory Key-Value / Data structure store
- **CAP Position:** AP (Redis Cluster), CA (single node)
- **အားသာချက်:** Sub-millisecond latency, rich data structures (String, Hash, List, Set, Sorted Set, Stream)
- **အားနည်းချက်:** Memory-bound (expensive at scale), persistence optional
- **သုံးသင့်သည့်အခါ:** Caching, session store, rate limiting, pub/sub, leaderboard, job queue
- **မသုံးသင့်သည့်အခါ:** Primary storage for large datasets, complex queries

### Amazon DynamoDB
- **အမျိုးအစား:** Key-Value + Document (NoSQL, managed)
- **CAP Position:** AP (eventually consistent) / CP (strongly consistent reads)
- **အားသာချက်:** Fully managed, auto-scaling, single-digit millisecond latency, global tables
- **အားနည်းချက်:** Vendor lock-in, query patterns must be pre-planned (no ad-hoc queries)
- **သုံးသင့်သည့်အခါ:** Serverless apps, gaming leaderboards, shopping carts, AWS-native systems
- **မသုံးသင့်သည့်အခါ:** Ad-hoc complex queries, joins, OLAP

### Apache Cassandra
- **အမျိုးအစား:** Wide-Column (NoSQL)
- **CAP Position:** AP (tunable consistency)
- **အားသာချက်:** Masterless architecture, linear horizontal scale, multi-region replication
- **အားနည်းချက်:** No joins, no transactions, data modeling ခက်ခဲ
- **သုံးသင့်သည့်အခါ:** Time-series IoT data, messaging platform, write-heavy workloads
- **မသုံးသင့်သည့်အခါ:** Complex queries, ACID transactions

### Elasticsearch
- **အမျိုးအစား:** Search Engine (Document-based)
- **CAP Position:** AP
- **အားသာချက်:** Full-text search, fuzzy matching, aggregations, log analytics (ELK Stack)
- **အားနည်းချက်:** Primary storage မဖြစ်သင့် (consistency ကြောင့်), updates costly
- **သုံးသင့်သည့်အခါ:** Search functionality, log/event analytics, e-commerce product search
- **မသုံးသင့်သည့်အခါ:** Primary data store, ACID transactions

### Neo4j
- **အမျိုးအစား:** Graph Database
- **CAP Position:** CA (single instance); CP (causal cluster)
- **အားသာချက်:** Relationship traversal O(1), Cypher query language, ACID
- **အားနည်းချက်:** Horizontal scaling ကန့်သတ်ချက်, not suited for tabular data
- **သုံးသင့်သည့်အခါ:** Social networks, fraud detection, knowledge graphs, recommendation engines
- **မသုံးသင့်သည့်အခါ:** Simple CRUD, high-volume tabular data

### InfluxDB
- **အမျိုးအစား:** Time-Series Database
- **CAP Position:** AP
- **အားသာချက်:** Time-series optimized writes/queries, retention policies, downsampling
- **အားနည်းချက်:** Time-series data သာ suitable, general queries အတွက် မသင့်
- **သုံးသင့်သည့်အခါ:** Infrastructure metrics, IoT sensor data, financial market data
- **မသုံးသင့်သည့်အခါ:** User data, product catalog, transactions

### CockroachDB
- **အမျိုးအစား:** NewSQL (Distributed Relational)
- **CAP Position:** CP
- **အားသာချက်:** PostgreSQL compatible, distributed ACID, auto-sharding, geo-distribution
- **အားနည်းချက်:** Higher latency than single-node Postgres, complexity
- **သုံးသင့်သည့်အခါ:** Global applications needing ACID + horizontal scale, fintech
- **မသုံးသင့်သည့်အခါ:** Read-heavy analytics, single-region deployments (overhead မခုံသာ)

---

## C.4 CAP Theorem Position — Database များ နေရာချ

```
         Consistency
              │
              │  PostgreSQL    CockroachDB
              │  MySQL         (CP/CA)
              │  (CA)
         CP ──┼────────────────────────
              │  MongoDB      Cassandra
              │  HBase        DynamoDB
              │  (CP)         (AP)
         AP ──┼──────────────────────
              │  CouchDB      Riak
              │              (AP)
              │
   Availability ────────── Partition Tolerance
```

| Database | CAP Classification | မှတ်ချက် |
|----------|------------------|---------|
| PostgreSQL | CA | Single node; partitioning မဖြစ်နိုင်ဟု assume |
| MySQL | CA | PostgreSQL နှင့် တူ |
| MongoDB | CP (default) | Primary node မှ read; strong consistency |
| Redis (Cluster) | AP | Split-brain possible |
| Cassandra | AP | Tunable (QUORUM ဖြင့် CP-like) |
| DynamoDB | AP (default) / CP | Strongly consistent reads = CP |
| CockroachDB | CP | Raft consensus |
| HBase | CP | ZooKeeper coordination |
| Elasticsearch | AP | Eventual consistency |
| Neo4j | CA (single) | Causal cluster = CP |

---

## C.5 Database ရွေးချယ်မှု Checklist

ရွေးချယ်ရန် မေးခွန်းများ —

- [ ] Data structure ဘာလဲ? (structured / semi-structured / unstructured)
- [ ] Read/Write ratio ဘယ်လောက်လဲ?
- [ ] Consistency requirement ဘယ်လောက် strict လဲ? (strong / eventual)
- [ ] Horizontal scaling လိုသလား?
- [ ] Query pattern (simple lookup / complex joins / full-text / time-range)?
- [ ] Transaction (ACID) လိုသလား?
- [ ] Latency requirement (millisecond / microsecond)?
- [ ] Cloud provider ကန့်သတ်ချက် ရှိသလား?
- [ ] Team ၏ expertise ဘာနှင့် ပိုကျွမ်းကျင်သလဲ?
- [ ] Operational overhead ဘယ်လောက် accept လုပ်နိုင်သလဲ?

---

## C.6 Polyglot Persistence ဥပမာ — E-commerce Platform

```
E-Commerce Platform
│
├── User accounts, orders        → PostgreSQL  (ACID, complex queries)
├── Product search               → Elasticsearch (full-text, facets)
├── Session / shopping cart      → Redis (fast, TTL-based)
├── Product catalog              → MongoDB (flexible schema, nested)
├── Recommendation engine        → Neo4j (user-product graph)
├── Analytics / reporting        → ClickHouse (OLAP, columnar)
├── Inventory time-series        → InfluxDB (time-series metrics)
└── CDN / static assets          → S3 / Blob Storage
```

> **သတိ:** Polyglot persistence သည် data consistency ကို ပိုရှုပ်ထွေးစေသည်။ Service ၏ boundary ကို ရှင်းလင်းစွာ သတ်မှတ်ပြီး event-driven synchronization သုံးပါ။
