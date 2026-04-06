# Microservices in Production: Architecture, Patterns & Real-World System Design

> **ဤစာအုပ်တစ်အုပ်လုံးကို Claude (Anthropic ၏ AI) က ရေးသားထားခြင်းဖြစ်ပြီး လူသားရေးသားသူ မပါဝင်ပါ။**
> အခန်းတိုင်း၊ code ဥပမာတိုင်း၊ architecture diagram တိုင်း နှင့် appendix တိုင်းကို Claude က generate ပြုလုပ်ထားသည်။ မြန်မာဘာသာ wording ပိုင်း၊ editorial review နှင့် final polish ကို Codex က ဆောင်ရွက်ထားသည်။ ဤ repository တွင် ပါဝင်သော စာသား (သို့) code တစ်ခုမှ လူသားက ရေးသားထားခြင်း မဟုတ်ပါ။

---

## ဤစာအုပ်အကြောင်း

Microservices architecture ကို **မြန်မာဘာသာ** ဖြင့် ပြည့်စုံစွာ ရေးသားထားသော စာအုပ်ဖြစ်သည်။ Communication pattern အားလုံး — synchronous (REST, gRPC, GraphQL)၊ asynchronous/event-driven (Kafka, Saga, CQRS, Event Sourcing) နှင့် real-time (WebSocket, SSE) — တို့ကို အကျုံးဝင်စွာ ဖော်ပြထားပြီး **production-scale system design case study ၂၃ ခု** ပါဝင်သည်။

### အဓိက အချက်အလက်များ

| အကြောင်းအရာ | ပမာဏ |
|-------------|-------|
| အခန်းအရေအတွက် | ၅၂ |
| Case Study အရေအတွက် | ၂၃ |
| Appendix အရေအတွက် | ၈ |
| စုစုပေါင်း စာကြောင်းအရေအတွက် | ~၂၈,၀၀၀+ |
| ခန့်မှန်း စကားလုံးအရေအတွက် | ~၂၂၀,၀၀၀+ |
| ဘာသာစကား | မြန်မာဘာသာ (English technical terms များ ပါဝင်) |
| ရေးသားသူ | Claude (Anthropic AI) |
| တည်းဖြတ်သူ / Reviewer | Codex |

### ပါဝင်သော အကြောင်းအရာများ

- မြန်မာဘာသာဖြင့် ရှင်းလင်းချက်များ၊ English technical terms များ မူရင်းအတိုင်း ထိန်းသိမ်းထားသည်
- Production-ready code ဥပမာများ (Python, Java, SQL, YAML, Protobuf, GraphQL)
- ASCII architecture diagram များ အခန်းတိုင်းတွင် ပါဝင်သည်
- နှိုင်းယှဉ်ဇယားများ နှင့် ဆုံးဖြတ်ချက်ချမှတ်ရေး လမ်းညွှန်များ
- အခန်းတိုင်း၏ အဆုံးတွင် အဓိကသင်ခန်းစာများ
- Interview ပြင်ဆင်ရေး playbook

---

## ဖိုင်ဖွဲ့စည်းပုံ

```
microservices/
  README.md                  # ဤဖိုင်
  chapters/
    part1/                   # အခြေခံသဘောတရားများ (အခန်း ၁–၃)
    part2/                   # Synchronous Communication (အခန်း ၄–၇)
    part3/                   # Event-Driven Communication (အခန်း ၈–၁၃)
    part4/                   # Real-Time Communication (အခန်း ၁၄–၁၅)
    part5/                   # Data Management (အခန်း ၁၆–၁၉)
    part6/                   # Infrastructure & Reliability (အခန်း ၂၀–၂၃)
    part7/                   # Deployment & Operations (အခန်း ၂၄–၂၆)
    part8/                   # Case Study: Sync-Heavy (အခန်း ၂၇–၃၀)
    part9/                   # Case Study: Event-Driven (အခန်း ၃၁–၃၅)
    part10/                  # Case Study: Real-Time (အခန်း ၃၆–၃၈)
    part11/                  # Case Study: Streaming & Media (အခန်း ၃၉–၄၁)
    part12/                  # Case Study: Social Platforms (အခန်း ၄၂–၄၃)
    part13/                  # Case Study: Location & Logistics (အခန်း ၄၄–၄၅)
    part14/                  # Case Study: Specialized (အခန်း ၄၆–၄၉)
    part15/                  # Advanced Topics (အခန်း ၅၀–၅၂)
    appendices/              # ကိုးကားချက်များ (A–H)
```

---

## မာတိကာ

### အပိုင်း ၁: Microservices အခြေခံများ

**[အခန်း ၁: Monolith မှ Microservices သို့](chapters/part1/chapter-01.md)**
- ၁.၁ Microservice ဆိုတာ ဘာလဲ
- ၁.၂ ဘယ်အချိန်မှာ ခွဲမလဲ — ဘယ်အချိန်မှာ မခွဲသင့်လဲ
- ၁.၃ Decomposition နည်းဗျူဟာများ (Business Capability, Domain DDD, Volatility)
- ၁.၄ Strangler Fig Pattern
- ၁.၅ Anti-Corruption Layer
- ၁.၆ Trade-offs: Complexity vs. Scalability vs. Team Autonomy

**[အခန်း ၂: Microservices အတွက် Domain-Driven Design](chapters/part1/chapter-02.md)**
- ၂.၁ Ubiquitous Language & Bounded Contexts
- ၂.၂ Aggregates, Entities, Value Objects
- ၂.၃ Context Mapping (Shared Kernel, Customer-Supplier, Conformist)
- ၂.၄ Strategic DDD → Service Boundaries
- ၂.၅ Event Storming Workshop

**[အခန်း ၃: Service Communication ရှုခင်းတစ်ခုလုံး](chapters/part1/chapter-03.md)**
- ၃.၁ Synchronous vs. Asynchronous vs. Real-Time — ဘယ်အချိန်မှာ ဘာသုံးမလဲ
- ၃.၂ Trade-off Triangle: Consistency, Availability, Coupling
- ၃.၃ Use Case အလိုက် Protocol ရွေးချယ်ခြင်း
- ၃.၄ Hybrid Architecture များ — လက်တွေ့ system အများစုတွင် သုံးမျိုးလုံး သုံးသည်
- ၃.၅ Communication Pattern ဆုံးဖြတ်ချက် Decision Tree

---

### အပိုင်း ၂: Synchronous Communication

**[အခန်း ၄: REST & HTTP](chapters/part2/chapter-04.md)**
- ၄.၁ RESTful API Design အခြေခံမူများ
- ၄.၂ Resource Naming, Versioning, HATEOAS
- ၄.၃ Pagination, Filtering, Sorting (Scale ကြီးချိန်)
- ၄.၄ HTTP/2 & HTTP/3 အကျိုးကျေးဇူးများ
- ၄.၅ OpenAPI / Swagger Contract-First Design
- ၄.၆ REST API များတွင် Idempotency Keys

**[အခန်း ၅: gRPC & Protocol Buffers](chapters/part2/chapter-05.md)**
- ၅.၁ Internal Services အတွက် gRPC ကို REST ထက် ဘာကြောင့် ရွေးသနည်း
- ၅.၂ Protocol Buffers — Schema Definition & Serialization
- ၅.၃ Unary, Server Streaming, Client Streaming, Bidirectional
- ၅.၄ gRPC Deadlines, Cancellation, Interceptors
- ၅.၅ Browser Client များအတွက် gRPC-Web
- ၅.၆ gRPC vs. REST နှိုင်းယှဉ်ဇယား

**[အခန်း ၆: GraphQL](chapters/part2/chapter-06.md)**
- ၆.၁ Queries, Mutations, Subscriptions
- ၆.၂ Schema Design & Type System
- ၆.၃ N+1 ပြဿနာ & DataLoader (Batching)
- ၆.၄ GraphQL Federation — Service အများကို ပေါင်းစည်းခြင်း
- ၆.၅ Backend for Frontend (BFF) Pattern
- ၆.၆ GraphQL vs. REST vs. gRPC — ဆုံးဖြတ်ချက်ချမှတ်ရေး လမ်းညွှန်

**[အခန်း ၇: API Gateway & Service Mesh](chapters/part2/chapter-07.md)**
- ၇.၁ API Gateway တာဝန်များ (Auth, Rate Limiting, Routing, SSL)
- ၇.၂ Gateway Pattern များ: Edge, BFF, Micro-Gateway
- ၇.၃ Kong, AWS API Gateway, Nginx, Envoy
- ၇.၄ Service Mesh — ဘာကို ဖြေရှင်းသနည်း (Istio, Linkerd)
- ၇.၅ Sidecar Proxy Pattern
- ၇.၆ East-West vs. North-South Traffic

---

### အပိုင်း ၃: Asynchronous & Event-Driven Communication

**[အခန်း ၈: Event-Driven Architecture အခြေခံများ](chapters/part3/chapter-08.md)**
- ၈.၁ Events vs. Commands vs. Queries
- ၈.၂ Event Notification vs. Event-Carried State Transfer
- ၈.၃ Domain Events vs. Integration Events vs. System Events
- ၈.၄ Event Schema Design & Backward Compatibility
- ၈.၅ Idempotency & At-Least-Once Delivery
- ၈.၆ Event-Driven vs. Request-Driven — Trade-offs

**[အခန်း ၉: Messaging Infrastructure](chapters/part3/chapter-09.md)**
- ၉.၁ Message Brokers vs. Event Streaming Platforms
- ၉.၂ Apache Kafka Deep Dive (Topics, Partitions, Offsets, Consumer Groups, Streams, Connect)
- ၉.၃ RabbitMQ — Exchanges, Queues, Bindings, Routing Keys
- ၉.၄ AWS SQS / SNS / EventBridge
- ၉.၅ Kafka vs. RabbitMQ vs. SQS — ဆုံးဖြတ်ချက် လမ်းညွှန်
- ၉.၆ Dead Letter Queues & Poison Messages
- ၉.၇ Backpressure & Flow Control

**[အခန်း ၁၀: Event Sourcing](chapters/part3/chapter-10.md)**
- ၁၀.၁ Event Sourcing ဆိုတာ ဘာလဲ
- ၁၀.၂ Event Store ဒီဇိုင်း
- ၁၀.၃ Events မှ State ပြန်တည်ဆောက်ခြင်း (Replay)
- ၁၀.၄ Performance အတွက် Snapshots
- ၁၀.၅ Temporal Queries
- ၁၀.၆ Event Sourcing + CQRS ပူးတွဲအသုံးပြုခြင်း

**[အခန်း ၁၁: CQRS — Command Query Responsibility Segregation](chapters/part3/chapter-11.md)**
- ၁၁.၁ Read နှင့် Write Models ခွဲခြားခြင်း
- ၁၁.၂ Read Model Projections
- ၁၁.၃ Eventual Consistency Trade-offs
- ၁၁.၄ Materialized Views & Denormalization
- ၁၁.၅ Events မှ Read Store ကို Sync ပြုလုပ်ခြင်း
- ၁၁.၆ Event Sourcing မပါဘဲ CQRS သုံးခြင်း

**[အခန်း ၁၂: Saga Pattern — Distributed Transactions](chapters/part3/chapter-12.md)**
- ၁၂.၁ Microservices တွင် Two-Phase Commit ဘာကြောင့် မအလုပ်လုပ်သနည်း
- ၁၂.၂ Choreography-Based Sagas (Event-Driven)
- ၁၂.၃ Orchestration-Based Sagas (Central Coordinator)
- ၁၂.၄ Compensating Transactions — Events ဖြင့် Rollback ပြုလုပ်ခြင်း
- ၁၂.၅ Saga State Machine ဒီဇိုင်း
- ၁၂.၆ Choreography vs. Orchestration — ဘယ်အချိန်မှာ ဘယ်ဟာ သုံးမလဲ

**[အခန်း ၁၃: Transactional Outbox & Change Data Capture](chapters/part3/chapter-13.md)**
- ၁၃.၁ Dual-Write ပြဿနာ
- ၁၃.၂ Transactional Outbox Pattern
- ၁၃.၃ Polling Publisher
- ၁၃.၄ Debezium ဖြင့် Transaction Log Tailing (CDC)
- ၁၃.၅ Idempotent Consumers အတွက် Inbox Pattern
- ၁၃.၆ Schema Registry & Avro / Protobuf Event Contracts

---

### အပိုင်း ၄: Real-Time Communication

**[အခန်း ၁၄: WebSocket & Persistent Connections](chapters/part4/chapter-14.md)**
- ၁၄.၁ HTTP Long Polling vs. SSE vs. WebSocket
- ၁၄.၂ WebSocket Handshake & Protocol
- ၁၄.၃ Connection သန်းပေါင်းများစွာ စီမံခန့်ခွဲခြင်း
- ၁၄.၄ Presence Detection & Heartbeats
- ၁၄.၅ WebSocket Servers Horizontal Scaling (Sticky Sessions, Redis Pub/Sub)
- ၁၄.၆ Reconnection Strategy & Message Ordering

**[အခန်း ၁၅: Server-Sent Events & Streaming APIs](chapters/part4/chapter-15.md)**
- ၁၅.၁ SSE Protocol & Browser Support
- ၁၅.၂ Use Cases: Live Feeds, Dashboards, Notifications
- ၁၅.၃ SSE vs. WebSocket ဆုံးဖြတ်ချက် လမ်းညွှန်
- ၁၅.၄ Chunked HTTP Responses
- ၁၅.၅ gRPC Server Streams ဖြင့် Streaming

---

### အပိုင်း ၅: Data Management Patterns

**[အခန်း ၁၆: Service တစ်ခုစီတွင် Database သီးခြားထားခြင်း](chapters/part5/chapter-16.md)**
- ၁၆.၁ Shared Database က Microservices ကို ဘာကြောင့် ဖျက်စီးသနည်း
- ၁၆.၂ Polyglot Persistence — အလုပ်အတွက် မှန်ကန်သော DB
- ၁၆.၃ Services များကြား Data Consistency
- ၁၆.၄ API Composition Pattern (Services များကြား Join ပြုလုပ်ရန်)

**[အခန်း ၁၇: Distributed Caching](chapters/part5/chapter-17.md)**
- ၁၇.၁ Cache-Aside, Write-Through, Write-Behind, Refresh-Ahead
- ၁၇.၂ Redis Deep Dive — Data Structures, Pub/Sub, Lua Scripts
- ၁၇.၃ Cache Stampede & Dog-Piling ကာကွယ်ခြင်း
- ၁၇.၄ Cache Invalidation နည်းဗျူဟာများ
- ၁၇.၅ Distributed Locks — Redlock Algorithm
- ၁၇.၆ CDN Caching — Edge vs. Origin

**[အခန်း ၁၈: Sharding, Partitioning & Replication](chapters/part5/chapter-18.md)**
- ၁၈.၁ Horizontal vs. Vertical Scaling
- ၁၈.၂ Sharding နည်းဗျူဟာများ (Hash, Range, Directory)
- ၁၈.၃ Consistent Hashing — Virtual Nodes
- ၁၈.၄ Read Replicas & Leader Election
- ၁၈.၅ Hot Spot ပြဿနာ & Rebalancing
- ၁၈.၆ Global Databases (CockroachDB, Spanner, PlanetScale)

**[အခန်း ၁၉: Search Infrastructure](chapters/part5/chapter-19.md)**
- ၁၉.၁ Elasticsearch Architecture (Index, Shard, Replica)
- ၁၉.၂ Inverted Index & Full-Text Search
- ၁၉.၃ Primary DB မှ Elasticsearch သို့ Data Sync (CDC)
- ၁၉.၄ Relevance Tuning (BM25, Vector Search)
- ၁၉.၅ Autocomplete & Type-Ahead (Scale ကြီးချိန်)
- ၁၉.၆ Elasticsearch vs. OpenSearch vs. Typesense

---

### အပိုင်း ၆: Infrastructure & Reliability

**[အခန်း ၂၀: Service Discovery & Load Balancing](chapters/part6/chapter-20.md)**
- ၂၀.၁ Client-Side vs. Server-Side Discovery
- ၂၀.၂ Service Registry (Consul, Eureka, etcd)
- ၂၀.၃ Load Balancing နည်းဗျူဟာများ
- ၂၀.၄ Kubernetes တွင် DNS-Based Discovery
- ၂၀.၅ Global Server Load Balancing (GeoDNS)

**[အခန်း ၂၁: Resilience Patterns](chapters/part6/chapter-21.md)**
- ၂၁.၁ Circuit Breaker (Hystrix, Resilience4j)
- ၂၁.၂ Retry with Exponential Backoff & Jitter
- ၂၁.၃ Bulkhead Pattern
- ၂၁.၄ Timeout & Deadline Propagation
- ၂၁.၅ Rate Limiting & Throttling (Token Bucket, Leaky Bucket, Sliding Window)
- ၂၁.၆ Graceful Degradation & Fallback နည်းဗျူဟာများ

**[အခန်း ၂၂: Observability (စောင့်ကြည့်နိုင်မှု)](chapters/part6/chapter-22.md)**
- ၂၂.၁ မဏ္ဍိုင် သုံးခု: Logs, Metrics, Traces
- ၂၂.၂ Distributed Tracing (OpenTelemetry, Jaeger, Zipkin)
- ၂၂.၃ Structured Logging & Correlation IDs
- ၂၂.၄ Metrics & Alerting (Prometheus, Grafana)
- ၂၂.၅ SLI, SLO, SLA — Error Budgets သတ်မှတ်ခြင်း
- ၂၂.၆ Profiling & Continuous Profiling (Pyroscope)

**[အခန်း ၂၃: Microservices တွင် Security](chapters/part6/chapter-23.md)**
- ၂၃.၁ Authentication vs. Authorization
- ၂၃.၂ OAuth2 & OpenID Connect (JWT, Opaque Tokens)
- ၂၃.၃ Service-to-Service Communication အတွက် mTLS
- ၂၃.၄ Secrets Management (Vault, AWS Secrets Manager)
- ၂၃.၅ Zero Trust Architecture
- ၂၃.၆ API Security (OWASP Top 10 for APIs)

---

### အပိုင်း ၇: Deployment & Operations

**[အခန်း ၂၄: Containers & Kubernetes](chapters/part7/chapter-24.md)**
- ၂၄.၁ Docker & Container အကောင်းဆုံး လက်တွေ့များ
- ၂၄.၂ Kubernetes Core Concepts (Pod, Deployment, Service, Ingress, ConfigMap)
- ၂၄.၃ Horizontal Pod Autoscaling (HPA) & Vertical Pod Autoscaling (VPA)
- ၂၄.၄ Microservices အတွက် Helm Charts
- ၂၄.၅ Stateful Services အတွက် StatefulSets
- ၂၄.၆ Kubernetes Namespaces & Multi-Tenancy

**[အခန်း ၂၅: Microservices အတွက် CI/CD](chapters/part7/chapter-25.md)**
- ၂၅.၁ Service တစ်ခုစီအတွက် သီးခြား Deploy Pipeline
- ၂၅.၂ Blue-Green & Canary Deployments
- ၂၅.၃ Feature Flags (LaunchDarkly, Unleash)
- ၂၅.၄ GitOps (ArgoCD / Flux)
- ၂၅.၅ Testing Pyramid: Unit → Integration → Contract (Pact) → E2E
- ၂၅.၆ Consumer-Driven Contract Testing

**[အခန်း ၂၆: Multi-Region & Disaster Recovery](chapters/part7/chapter-26.md)**
- ၂၆.၁ Active-Active vs. Active-Passive
- ၂၆.၂ Regions များကြား Data Replication
- ၂၆.၃ RTO vs. RPO — Recovery Planning
- ၂၆.၄ Chaos Engineering (Chaos Monkey, Gremlin)
- ၂၆.၅ Runbooks & Game Days

---

### အပိုင်း ၈: Case Study များ — Synchronous-Heavy Systems

> *အဓိက pattern: REST/gRPC request-response။ Read-heavy, write complexity နည်း။*

- **[အခန်း ၂၇: URL Shortener (TinyURL / Bitly)](chapters/part8/chapter-27.md)**
- **[အခန်း ၂၈: Cloud Storage System (Dropbox / Google Drive)](chapters/part8/chapter-28.md)**
- **[အခန်း ၂၉: Authentication & Identity System (Okta / Auth0)](chapters/part8/chapter-29.md)**
- **[အခန်း ၃၀: Search Engine (Google / Elasticsearch-Based)](chapters/part8/chapter-30.md)**

---

### အပိုင်း ၉: Case Study များ — Event-Driven Systems

> *အဓိက pattern: Async messaging, Saga, Event Sourcing, CQRS။*

- **[အခန်း ၃၁: Banking System (ဘဏ်စနစ်)](chapters/part9/chapter-31.md)**
- **[အခန်း ၃၂: Payment Gateway System (Stripe / Razorpay)](chapters/part9/chapter-32.md)**
- **[အခန်း ၃၃: Digital Wallet System (PayPal / Paytm)](chapters/part9/chapter-33.md)**
- **[အခန်း ၃၄: E-Commerce Order System (Amazon)](chapters/part9/chapter-34.md)**
- **[အခန်း ၃၅: Food Delivery System (DoorDash / Uber Eats)](chapters/part9/chapter-35.md)**

---

### အပိုင်း ၁၀: Case Study များ — Real-Time Systems

> *အဓိက pattern: WebSocket, persistent connections, low-latency state sync။*

- **[အခန်း ၃၆: Chat & Messaging System (WhatsApp / Slack)](chapters/part10/chapter-36.md)**
- **[အခန်း ၃၇: Stock Trading System (ရှယ်ယာအရောင်းအဝယ်စနစ်)](chapters/part10/chapter-37.md)**
- **[အခန်း ၃၈: Live Sports & Gaming Leaderboard](chapters/part10/chapter-38.md)**

---

### အပိုင်း ၁၁: Case Study များ — Streaming & Media

> *အဓိက pattern: Async pipelines, CDN, adaptive streaming။*

- **[အခန်း ၃၉: Video Streaming System (Netflix)](chapters/part11/chapter-39.md)**
- **[အခန်း ၄၀: Music Streaming System (Spotify)](chapters/part11/chapter-40.md)**
- **[အခန်း ၄၁: Video Upload & Sharing Platform (YouTube)](chapters/part11/chapter-41.md)**

---

### အပိုင်း ၁၂: Case Study များ — Social & Content Platforms

> *အဓိက pattern: Hybrid — fan-out on write/read, graph traversal, feed generation။*

- **[အခန်း ၄၂: Social Media Feed (Twitter / X)](chapters/part12/chapter-42.md)**
- **[အခန်း ၄၃: Photo Sharing Platform (Instagram)](chapters/part12/chapter-43.md)**

---

### အပိုင်း ၁၃: Case Study များ — Location & Logistics

> *အဓိက pattern: Real-time geospatial, event-driven dispatch, ML integration။*

- **[အခန်း ၄၄: Ride-Sharing System (Uber / Lyft)](chapters/part13/chapter-44.md)**
- **[အခန်း ၄၅: Maps & Navigation System (Google Maps)](chapters/part13/chapter-45.md)**

---

### အပိုင်း ၁၄: Case Study များ — Infrastructure & Specialized Systems

> *အဓိက pattern: ကွဲပြားသည် — scale, latency နှင့် correctness အပေါ် အာရုံစိုက်သည်။*

- **[အခန်း ၄၆: Notification System (အကြောင်းကြားစနစ်)](chapters/part14/chapter-46.md)**
- **[အခန်း ၄၇: Ad Serving System (Google Ads / Meta Ads)](chapters/part14/chapter-47.md)**
- **[အခန်း ၄၈: Healthcare System (EMR / Telemedicine)](chapters/part14/chapter-48.md)**
- **[အခန်း ၄၉: Rate Limiter as a Service](chapters/part14/chapter-49.md)**

---

### အပိုင်း ၁၅: Advanced Topics

- **[အခန်း ၅၀: Event-Driven Patterns (Scale ကြီးချိန်)](chapters/part15/chapter-50.md)**
- **[အခန်း ၅၁: Microservices Anti-Patterns (ရှောင်ကြဉ်ရမည့် ပုံစံများ)](chapters/part15/chapter-51.md)**
- **[အခန်း ၅၂: Microservices Interview Playbook (အင်တာဗျူး ပြင်ဆင်ရေး)](chapters/part15/chapter-52.md)**

---

### နောက်ဆက်တွဲများ (Appendices)

| နောက်ဆက်တွဲ | ခေါင်းစဉ် |
|-------------|----------|
| A | System Design ခန့်မှန်းချက် အမြန်ကိုးကားကဒ် |
| B | Kafka vs. RabbitMQ vs. SQS vs. Pulsar နှိုင်းယှဉ်ဇယား |
| C | Database ရွေးချယ်ရေး လမ်းညွှန် (SQL vs. NoSQL vs. NewSQL) |
| D | CAP Theorem & PACELC ကိုးကားချက် |
| E | API Protocol နှိုင်းယှဉ်ချက် — REST vs. gRPC vs. GraphQL vs. WebSocket |
| F | Saga Pattern Templates (Choreography & Orchestration) |
| G | Architecture Diagram သင်္ကေတများ & ဓလေ့ထုံးတမ်းများ |
| H | ဝေါဟာရ အဓိပ္ပာယ်ဖွင့်ဆိုချက်များ (Glossary) |

---

## အနှစ်ချုပ်

| အပိုင်း | အာရုံစိုက်မှု | အခန်းများ |
|---------|-------------|----------|
| ၁ | အခြေခံ & DDD | ၁–၃ |
| ၂ | Synchronous (REST, gRPC, GraphQL) | ၄–၇ |
| ၃ | Async & Event-Driven | ၈–၁၃ |
| ၄ | Real-Time (WebSocket, SSE) | ၁၄–၁၅ |
| ၅ | Data Management | ၁၆–၁၉ |
| ၆ | Infrastructure & Reliability | ၂၀–၂၃ |
| ၇ | Deployment & Operations | ၂၄–၂၆ |
| ၈ | Case Study: Synchronous-Heavy | ၂၇–၃၀ |
| ၉ | Case Study: Event-Driven | ၃၁–၃၅ |
| ၁၀ | Case Study: Real-Time | ၃၆–၃၈ |
| ၁၁ | Case Study: Streaming & Media | ၃၉–၄၁ |
| ၁၂ | Case Study: Social Platforms | ၄၂–၄၃ |
| ၁၃ | Case Study: Location & Logistics | ၄၄–၄၅ |
| ၁၄ | Case Study: Specialized Systems | ၄၆–၄၉ |
| ၁၅ | Advanced Topics & Interview ပြင်ဆင်ရေး | ၅၀–၅၂ |

**စုစုပေါင်း: အခန်း ၅၂ ခု | Case Study ၂၃ ခု | Appendix ၈ ခု | စကားလုံး ~၂၂၀,၀၀၀+**

---

## ရေးသားသူနှင့် တည်းဖြတ်သူအကြောင်း ရှင်းလင်းချက်

ဤစာအုပ်ကို **Claude** ([Anthropic](https://www.anthropic.com) ၏ AI assistant) က **အပြည့်အဝ ရေးသားထားခြင်း** ဖြစ်သည်။ လူသားရေးသားသူ မည်သူမျှ ပါဝင်ခဲ့ခြင်း မရှိပါ။ အကြောင်းအရာရေးသားခြင်းကို Claude က လုပ်ဆောင်ထားပြီး **Codex** က Burmese wording, editorial review နှင့် final editing ပိုင်းကို ဆောင်ရွက်ထားသည်။ ဤစာအုပ်ကို ၂၀၂၆ ခုနှစ် ဧပြီလတွင် Claude models များဖြင့် generate ပြုလုပ်ထားသည်။

**ဆိုလိုရင်းမှာ:**
- စာသား၊ code ဥပမာ၊ diagram နှင့် technical ရှင်းလင်းချက် အားလုံးကို AI က ထုတ်လုပ်ထားသည်
- အကြောင်းအရာများသည် Claude ၏ training data နှင့် reasoning ပေါ် အခြေခံပြီး ပထမလက် engineering အတွေ့အကြုံ မဟုတ်ပါ
- Technical claims များ (scale ဂဏန်းများ၊ architecture ဆုံးဖြတ်ချက်များ) သည် အများသိ pattern များ ပေါ်တွင် အခြေခံပြီး ကိုးကား company များ၏ တကယ့် production implementation ကို တိကျစွာ ထင်ဟပ်ချင်မှ ထင်ဟပ်နိုင်သည်
- Production တွင် အသုံးမချမီ အရေးကြီးသော technical အချက်အလက်များကို တရားဝင် documentation ဖြင့် စစ်ဆေးအတည်ပြုသင့်သည်

**တည်ဆောက်ပုံ:**
- မာတိကာကို scope နှင့် ပါဝင်ရမည့် system များ သတ်မှတ်ပေးသော လူသားနှင့် ပူးပေါင်း ဒီဇိုင်းချထားသည်
- အခန်း ၅၂ ခု + appendix ၈ ခုကို Claude က အဓိက ရေးသားထားပြီး Codex က review/edit လုပ်ငန်းစဉ်ကို ပံ့ပိုးထားသည်
- Editorial review အဆင့်များတွင် Codex က Burmese phrasing, formatting drift နှင့် placeholder စာသားများကို စစ်ဆေးတည်းဖြတ်ထားသည်

---

## လိုင်စင်

ဤစာအုပ်သည် open educational resource တစ်ခုဖြစ်သည်။ သင်ယူခြင်း၊ ကိုးကားခြင်း နှင့် သင်ကြားခြင်းအတွက် အသုံးပြုနိုင်ပါသည်။
