# အခန်း ၂၇: URL Shortener စနစ် (TinyURL / Bitly)

## နိဒါန်း

URL Shortener စနစ်သည် ခေတ်မီ web application များ၌ အသုံးများသော အခြေခံ system design ဥပမာတစ်ခုဖြစ်သည်။ TinyURL နှင့် Bitly ကဲ့သို့သော ဝန်ဆောင်မှုများသည် ရှည်လျားသော URL များကို တိုတောင်းသော အက္ခရာများဖြင့် ဖေါ်ပြ၍ share ပြုလုပ်ရလွယ်ကူစေသည်။ ဤအခန်းတွင် ၎င်းစနစ်ကို production-grade microservices architecture ဖြင့် မည်ကဲ့သို့ တည်ဆောက်မည်ကို အသေးစိတ် လေ့လာသွားမည်ဖြစ်သည်။

URL Shortener ၏ ရည်ရွယ်ချက်မှာ ရိုးရှင်းသည် — long URL တစ်ခုကို short URL တစ်ခုသို့ map ပြုလုပ်ပေးပြီး user သည် short URL သို့ access ပြုလုပ်သောအခါ original URL သို့ redirect ပြုလုပ်ပေးသည်။ သို့သော် ဤလုပ်ငန်းစဉ်ကို billions of requests ကို handle ပြုလုပ်နိုင်သော scale တွင် ဆောင်ရွက်ရန်မှာ engineering challenge များစွာ ပါဝင်သည်။

---

## ၂၇.၁ Requirements နှင့် Scale Estimation

### Functional Requirements

- **URL Shortening**: Long URL တစ်ခုကို input ရရှိ၍ unique short URL တစ်ခု generate ပြုလုပ်ပေးရမည်။
- **URL Redirection**: Short URL သို့ HTTP request ရရှိသောအခါ original URL သို့ redirect ပြုလုပ်ရမည်။
- **Custom Alias**: User သည် မိမိနှစ်သက်သော custom short code ကို သတ်မှတ်နိုင်ရမည်။
- **Expiry**: URL များကို expiry date ဖြင့် သတ်မှတ်နိုင်ရမည်။
- **Analytics**: Click count၊ geographic data နှင့် device type စသည့် analytics data ကို ကောက်ယူနိုင်ရမည်။

### Non-Functional Requirements

- **High Availability**: 99.99% uptime (downtime တစ်နှစ်လျှင် မိနစ် ၅၂ ခန့်သာ)
- **Low Latency**: Redirect latency < 10ms (P99)
- **Scalability**: Horizontal scaling ဖြင့် load ကို handle ပြုလုပ်နိုင်ရမည်။
- **Durability**: Data loss မဖြစ်ရမည်။

### Scale Estimation

```
Write (Shortening) Operations:
  - 100 million URLs/day
  - 100M / 86,400s ≈ 1,157 writes/second (peak: ~3,000 writes/s)

Read (Redirect) Operations:
  - Read:Write ratio = 10:1
  - 1,000M redirects/day
  - ≈ 11,574 reads/second (peak: ~30,000 reads/s)

Storage Estimation:
  - Average URL size: 500 bytes
  - Short code + metadata: 300 bytes
  - Total per URL: ~800 bytes
  - 100M URLs/day × 365 days × 5 years = 182.5 billion URLs
  - 182.5B × 800 bytes ≈ 146 TB (5 years)

Cache Estimation:
  - 80/20 rule: Top 20% URLs = 80% traffic
  - Daily reads: 1B × 20% = 200M URLs cached
  - 200M × 800 bytes ≈ 160 GB cache
```

---

## ၂၇.၂ Domain Decomposition

URL Shortener system ကို microservices သုံးခုဖြင့် ပိုင်းခြားသည်။

### ၂၇.၂.၁ Shortening Service

Long URL ကို short URL သို့ ပြောင်းလဲပေးသော service ဖြစ်သည်။ ၎င်းသည် hash generation၊ collision detection နှင့် database write operation များကို တာဝန်ယူသည်။

**Responsibilities:**
- URL validation (malformed URL၊ malware detection)
- Short code generation
- Custom alias validation
- Expiry date management
- Database write

### ၂၇.၂.၂ Redirect Service

Short URL request ကို receive ပြုလုပ်၍ original URL သို့ redirect ပြုလုပ်ပေးသော service ဖြစ်သည်။ ဤ service သည် latency-critical ဖြစ်ပြီး Redis cache ကို heavily depend ပြုသည်။

**Responsibilities:**
- Short code lookup (Cache → Database)
- HTTP redirect response (301 or 302)
- Analytics event publish
- Expiry check

### ၂၇.၂.၃ Analytics Service

Click events များကို process ပြုလုပ်၍ dashboard အတွက် aggregated data ကို ပေးသော service ဖြစ်သည်။

**Responsibilities:**
- Event consumption from Kafka
- Geolocation lookup (IP → Country/City)
- Device type parsing (User-Agent)
- Time-series aggregation
- Dashboard API

---

## ၂၇.၃ Hash Generation Strategies

Short code generation strategy သည် system ၏ correctness နှင့် performance ကို တိုက်ရိုက်သက်ရောက်သည်။

### MD5 / SHA-256 Hashing

Long URL ကို MD5 hash ပြုလုပ်၍ ပထမ characters ၇ ခုကို ယူသည်။

```
md5("https://www.example.com/very-long-path") = "a9993e36..."
short_code = "a9993e3" (first 7 chars)
```

**ပြဿနာ:** Collision probability မြင့်သည်။ 7-character Base16 = 268 million combinations သာ ရှိသည်။ 100M URLs ကြောင့် birthday paradox ဖြင့် collision ဖြစ်နိုင်သည်။

### Base62 Encoding

`[0-9][a-z][A-Z]` = 62 characters ဖြင့် counter-based ID ကို encode ပြုလုပ်သည်။

```
ID: 123456789
Base62: "8M0kX" (5 chars)
Base62 with 7 chars = 62^7 = 3.5 trillion combinations
```

**အားသာချက်:** No collision၊ predictable length၊ fast generation

**အားနည်းချက်:** ID sequential ဖြစ်ပါက enumerate attack ဖြစ်နိုင်သည်။ ဖြေရှင်းချက် — ID ကို XOR သို့မဟုတ် hash ပြုလုပ်ပြီးမှ encode လုပ်သည်။

### Counter-Based with Distributed ID Generator

**Snowflake-style ID** (Twitter Snowflake pattern) ဖြင့် globally unique, time-sorted ID generate ပြုလုပ်၍ Base62 encode လုပ်သည်။

```
[41-bit timestamp][10-bit machine_id][12-bit sequence]
= 64-bit integer → Base62 → "dZV1mn2"
```

**production recommended:** ဤ approach သည် collision-free, distributed-safe နှင့် sortable ဖြစ်သည်။

---

## ၂၇.၄ Database Design — KV Store (Redis + Cassandra)

### Primary Storage: Cassandra

```sql
CREATE TABLE url_mappings (
    short_code  TEXT PRIMARY KEY,
    long_url    TEXT,
    user_id     UUID,
    created_at  TIMESTAMP,
    expires_at  TIMESTAMP,
    is_custom   BOOLEAN,
    is_active   BOOLEAN
);

CREATE TABLE url_by_user (
    user_id    UUID,
    created_at TIMESTAMP,
    short_code TEXT,
    long_url   TEXT,
    PRIMARY KEY ((user_id), created_at, short_code)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

**Cassandra ကို ရွေးချယ်ရသော အကြောင်းရင်း:**
- Write-heavy workload ကို ကောင်းစွာ handle ပြုနိုင်သည်
- Linear horizontal scaling
- High availability with tunable consistency
- Time-series data (analytics) အတွက် suitable

### Cache Layer: Redis

```
Key: short_code (e.g., "dZV1mn2")
Value: {long_url, expires_at, is_active}
TTL: 24 hours (hot URLs) / 1 hour (cold URLs)
```

**Cache Strategy: Cache-Aside**
1. Request ရရှိသောအခါ Redis ကို အရင် query
2. Cache miss ဖြစ်ပါက Cassandra ကို query
3. Result ကို Redis တွင် cache လုပ်သည်
4. Subsequent requests တွင် Redis မှ serve

**Cache Eviction:** LRU (Least Recently Used) policy

---

## ၂၇.၅ 301 vs 302 Redirect — Analytics နှင့် Caching ကို သက်ရောက်မှု

### 301 Permanent Redirect

```http
HTTP/1.1 301 Moved Permanently
Location: https://original-url.com/path
Cache-Control: max-age=31536000
```

**ဂုဏ်သတ္တိများ:**
- Browser သည် result ကို cache လုပ်ထားသည်
- Subsequent requests တွင် server သို့ မရောက်တော့ပါ
- **Analytics အတွက် မကောင်း** — click count မတိကျ
- Server load နည်းသည်

### 302 Temporary Redirect

```http
HTTP/1.1 302 Found
Location: https://original-url.com/path
Cache-Control: no-cache
```

**ဂုဏ်သတ္တိများ:**
- Browser သည် cache မလုပ်
- Every click သည် server သို့ ရောက်သည်
- **Analytics အတွက် ကောင်း** — accurate click tracking
- Server load ပိုများသည်

### Design Decision

Analytics ကို priority ထားသော system တွင် **302** ကို default အဖြစ် သုံးသည်။ User ၏ privacy settings ပေါ်မူတည်၍ 301 ကို opt-in အဖြစ် ပေးနိုင်သည်။

---

## ၂၇.၆ Custom Alias နှင့် Expiry စီမံခန့်ခွဲမှု

### Custom Alias

```
User input: "my-brand"
Validation: 
  - Length: 3-30 characters
  - Allowed: [a-zA-Z0-9-_]
  - Not reserved: ["api", "admin", "login", ...]
  - Not taken: Check in Cassandra

Conflict Resolution:
  - If taken: Return 409 Conflict
  - Suggest alternatives: "my-brand-1", "my-brand-2"
```

### Expiry Management

```
URL creation time: T0
Expiry options:
  - 1 hour, 1 day, 1 week, 1 month, custom date
  - Default: Never expires (unless system policy)

Expiry check flow:
  1. Redis stores expires_at in value
  2. Redirect Service checks expires_at
  3. If expired: Return 410 Gone
  4. Background job: Cassandra TTL cleanup (soft delete first)
```

---

## ၂၇.၇ Bloom Filter ဖြင့် Duplicate Detection

Bloom Filter သည် probabilistic data structure တစ်ခုဖြစ်ပြီး element တစ်ခု set ထဲတွင် ရှိ/မရှိ ကို space-efficient ကြောင့် စစ်ဆေးနိုင်သည်။

### URL Deduplication

Long URL တစ်ခု existing short code ရှိ/မရှိ စစ်ဆေးရာတွင် သုံးသည်။

```
Flow:
1. New long_url ရရှိ
2. Bloom Filter check: "Definitely NOT in DB" → New URL → Generate short code
3. "POSSIBLY in DB" → Database check (false positive possible)
4. Database hit → Return existing short code
5. Database miss → Generate new short code

Memory: 100M URLs × 10 bits/element ≈ 125 MB
False positive rate: ~1% (acceptable)
```

### Short Code Uniqueness

```
Generated short_code → Bloom Filter check → 
  Not exists → Insert to DB + Bloom Filter
  Possibly exists → DB check → 
    Collision → Regenerate with salt
    No collision → Insert to DB + Bloom Filter
```

---

## ၂၇.၈ Click Analytics Pipeline (Async Event-Driven)

Analytics pipeline သည် redirect performance ကို မထိခိုက်ရန် async ဖြင့် ဆောင်ရွက်သည်။

### Event Publishing

```
Redirect Service → Kafka Topic: "click-events"

Event Schema:
{
  "short_code": "dZV1mn2",
  "timestamp": "2026-04-06T10:00:00Z",
  "ip_address": "203.0.113.42",
  "user_agent": "Mozilla/5.0 (iPhone...)",
  "referer": "https://twitter.com/...",
  "country": "MM",  // resolved by GeoIP
  "device_type": "mobile"
}
```

### Analytics Processing Pipeline

```
Kafka → Analytics Consumer (Flink/Spark Streaming)
  ├── Enrich: IP → GeoIP lookup → Country/City
  ├── Parse: User-Agent → Device/Browser/OS
  ├── Aggregate: 
  │   ├── clicks_by_hour (time-series)
  │   ├── clicks_by_country
  │   └── clicks_by_device
  └── Sink: 
      ├── ClickHouse (time-series analytics DB)
      └── Redis (real-time counters)
```

### Real-Time Counter

```
Redis INCR operations:
  INCR clicks:dZV1mn2:total
  INCR clicks:dZV1mn2:2026-04-06
  INCR clicks:dZV1mn2:country:MM
```

---

## ၂၇.၉ Architecture Diagram နှင့် Data Flow

### System Architecture

```
                    ┌─────────────────────────────────────────────────┐
                    │                   Client Layer                   │
                    │         Browser / Mobile App / API Client        │
                    └──────────────────┬──────────────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────────────┐
                    │              API Gateway / Load Balancer         │
                    │         (Rate Limiting, Auth, SSL Termination)   │
                    └──────┬───────────────────────────┬──────────────┘
                           │                           │
          ┌────────────────▼──────────┐   ┌────────────▼──────────────┐
          │    Shortening Service     │   │    Redirect Service        │
          │  (URL validation,         │   │  (Cache lookup,            │
          │   Hash generation,        │   │   DB fallback,             │
          │   DB write)               │   │   Analytics publish)       │
          └────────────────┬──────────┘   └────────────┬──────────────┘
                           │                           │
          ┌────────────────▼───────────────────────────▼──────────────┐
          │                    Data Layer                               │
          │  ┌─────────────────┐    ┌─────────────────────────────┐   │
          │  │  Redis Cluster  │    │    Cassandra Cluster         │   │
          │  │  (Hot URL Cache)│    │    (Persistent Storage)      │   │
          │  │  160 GB         │    │    146 TB (5 years)          │   │
          │  └─────────────────┘    └─────────────────────────────┘   │
          └──────────────────────────────────────────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────────────┐
                    │            Analytics Pipeline                     │
                    │  Kafka → Flink Streaming → ClickHouse + Redis    │
                    └──────────────────┬──────────────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────────────┐
                    │           Analytics Service                      │
                    │        (Dashboard API, Reports)                  │
                    └─────────────────────────────────────────────────┘
```

### URL Shortening Flow

```
1. POST /api/shorten {long_url: "https://..."}
2. API Gateway → Rate limit check → Auth check
3. Shortening Service:
   a. URL validation (format, malware check)
   b. Bloom Filter: Check duplicate
   c. Generate short_code (Base62 + Snowflake ID)
   d. Write to Cassandra (primary write)
   e. Write to Redis (cache warm-up)
   f. Return: {short_url: "https://short.ly/dZV1mn2"}

4. User shares short URL

5. GET /dZV1mn2
6. Redirect Service:
   a. Extract short_code: "dZV1mn2"
   b. Redis lookup: HIT → Get long_url
   c. Check expiry: Valid
   d. Publish click event to Kafka (async, fire-and-forget)
   e. Return HTTP 302 Location: "https://original-url.com"

7. Kafka Consumer (Analytics Service):
   a. Consume click event
   b. Enrich with GeoIP + User-Agent parsing
   c. Write to ClickHouse (batch)
   d. INCR Redis counters (real-time)
```

---

## ၂၇.၁၀ Key Design Decisions နှင့် Trade-offs

| Design Decision | ရွေးချယ်ခဲ့သည် | Alternative | Trade-off |
|---|---|---|---|
| ID Generation | Snowflake + Base62 | MD5 Hash | Collision-free vs Simpler |
| Cache | Redis Cluster | Memcached | Rich data types vs Simpler |
| DB | Cassandra | MySQL | Write scalability vs ACID |
| Redirect | 302 | 301 | Analytics accuracy vs Server load |
| Analytics | Async Kafka | Sync DB write | Performance vs Consistency |
| Bloom Filter | Yes | DB-only check | Memory efficiency vs Simplicity |

### Bottleneck Analysis

**Read Path (Hot Path):**
- Redis cache hit ratio ≥ 95% ကို maintain ရမည်
- Redis cluster sharding ဖြင့် horizontal scale
- CDN edge nodes တွင် popular URLs ကို cache (< 5ms latency)

**Write Path:**
- Cassandra multi-datacenter replication
- Async write ဖြင့် latency ကို minimize
- Backpressure mechanism ဖြင့် overflow handle

### Failure Scenarios

1. **Redis down**: Cassandra fallback၊ latency ပိုများ (50ms → 200ms)
2. **Cassandra down**: Write buffer to Kafka၊ eventual consistency
3. **Kafka lag**: Analytics delayed, redirect unaffected
4. **Bloom Filter false positive**: Extra DB read၊ correctness intact

---

## အနှစ်ချုပ်

URL Shortener system သည် simple ဟုထင်ရသော်လည်း production scale တွင် caching strategy, hash collision avoidance, analytics pipeline design, နှင့် redirect semantics (301 vs 302) စသည့် engineering decisions များ ပါဝင်သည်။ Redis + Cassandra combination ဖြင့် hot read path ကို optimize ပြုလုပ်ပြီး Kafka-based async analytics ဖြင့် redirect latency ကို မထိခိုက်ဘဲ rich analytics data ကောက်ယူနိုင်သည်။ ဤ system ၏ core challenge မှာ 30,000+ reads/second ကို sub-10ms latency ဖြင့် handle ပြုလုပ်ရင်း analytics accuracy ကို မဆုံးရှုံးရသော balance ကို ရှာဖွေခြင်းဖြစ်သည်။
