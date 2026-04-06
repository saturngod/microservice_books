# နောက်ဆက်တွဲ H: အဓိပ္ပာယ်ဖွင့်ဆိုချက်များ (Glossary of Terms)

> **ဖတ်ရှုမှုလမ်းညွှန်:** ဤ glossary တွင် technical term များကို English တွင် ဖော်ပြပြီး မြန်မာဘာသာဖြင့် ရှင်းလင်းချက် (2-3 ကြောင်း) ပါဝင်သည်။ Term များကို A–Z အစဉ်အတိုင်း စီစဉ်ထားသည်။

---

## A

### API Gateway
API Gateway သည် client requests အားလုံးကို receive ပြီး backend microservices သို့ route ပေးသော single entry point ဖြစ်သည်။ Authentication, rate limiting, SSL termination, request routing ကဲ့သို့သော cross-cutting concerns များကို ဗဟိုချက်တစ်ခုတည်းတွင် handle ပြုလုပ်သည်။ ဥပမာများ — Kong, AWS API Gateway, Nginx, Traefik ။

### Avro
Apache Avro သည် data serialization framework ဖြစ်ပြီး JSON-based schema definition ဖြင့် compact binary format ဖြစ်အောင် data ကို serialize/deserialize ပြုသည်။ Schema ကို data နှင့်အတူ (.avro file header တွင်) သိမ်းဆည်းသောကြောင့် schema evolution (backward/forward compatibility) ကို native ဖြင့် support ပြုပြီး Kafka ecosystem တွင် Schema Registry နှင့် တွဲဖက်သုံးလေ့ရှိသည်။ Protobuf နှင့် ယှဉ်လျှင် Avro သည် dynamic schema resolution ကို support ပြုပြီး code generation မလိုအပ်ဘဲ data ကို process ပြုနိုင်သောကြောင့် data pipeline / analytics use cases တွင် popular ဖြစ်သည်။

---

## B

### Backpressure
Backpressure သည် system ၏ downstream component (consumer) သည် upstream component (producer) ထက် processing speed နှေးသောအခါ producer ကို slow down ပြုရန် signal ပေးသော flow control mechanism ဖြစ်သည်။ Backpressure မရှိပါက unbounded buffer growth (memory exhaustion), message drop, system crash တို့ ဖြစ်နိုင်သည်။ Reactive Streams specification (Java), Kafka consumer lag monitoring, TCP flow control, gRPC flow control window တို့သည် backpressure implementation ၏ ဥပမာများဖြစ်ပြီး message queue depth alerting နှင့် consumer auto-scaling ကိုလည်း backpressure handling strategy အဖြစ် အသုံးပြုနိုင်သည်။

### Bloom Filter
Bloom Filter သည် element တစ်ခု set ထဲတွင် ရှိ/မရှိ စစ်ဆေးနိုင်သော probabilistic data structure ဖြစ်သည်။ False positive ဖြစ်နိုင်ပြီး (ရှိသည်ဟု မှားဖြေနိုင်) false negative မဖြစ်နိုင် (မရှိဟု မမှားဖြေ)။ Cache lookup optimization, URL blocklist check, database query skip တို့တွင် memory-efficient ဖြေရှင်းချက်အဖြစ် အသုံးပြုသည်။

### Bounded Context
Bounded Context သည် Domain-Driven Design (DDD) မှ concept ဖြစ်ပြီး business domain ၏ model တစ်ခု valid ဖြစ်သည့် boundary ကို သတ်မှတ်သည်။ Bounded Context တစ်ခုအတွင်း vocabulary (ubiquitous language) နှင့် rules တို့ consistent ဖြစ်ရမည်။ Microservices တွင် Bounded Context တစ်ခုသည် service တစ်ခု (သို့မဟုတ် service အနည်းငယ်) ၏ boundary ကို သတ်မှတ်ရာတွင် အသုံးဝင်သည်။

---

## C

### CAP Theorem
CAP Theorem (Brewer's Theorem) သည် distributed system တစ်ခုသည် Consistency, Availability, Partition Tolerance ဟူသော ဂုဏ်သတ္တိသုံးခုထဲမှ တစ်ချိန်တည်း နှစ်ခုသာ guarantee ပေးနိုင်သည်ဟု ဆိုသည်။ Network partition မည်သည်တိုင် ဖြစ်တတ်သောကြောင့် distributed system များသည် C နှင့် A ၏ trade-off ကိုသာ ရွေးချယ်ရသည်ဟု လက်တွေ့တွင် ရှင်းပြကြသည်။ CP systems များသည် data correctness ကို ဦးစားပေးပြီး AP systems များသည် availability ကို ဦးစားပေးသည်။

### CDC (Change Data Capture)
CDC (Change Data Capture) သည် database ၏ row-level changes (INSERT, UPDATE, DELETE) ကို real-time တွင် capture ပြီး downstream systems သို့ stream ပေးသော technique ဖြစ်သည်။ Application code မပြောင်းဘဲ database log (transaction log / WAL) ကို read ဖြင့် အလုပ်လုပ်သည်။ Debezium, AWS DMS, Maxwell ကဲ့သို့ tools များဖြင့် implement လုပ်ပြီး event sourcing, data synchronization, cache invalidation တို့တွင် သုံးသည်။

### Compensating Transaction
Compensating Transaction သည် distributed transaction (Saga) တွင် step တစ်ခု fail ဖြစ်သည့်အခါ ယခင် successfully completed steps များ၏ effects ကို semantically undo ပြုလုပ်သော reverse operation ဖြစ်သည်။ Database rollback ကဲ့သို့ exact undo မဟုတ်ဘဲ business logic level ဖြင့် correction ပြုလုပ်ခြင်းဖြစ်သည် (e.g., payment debit ကို compensate ရန် refund credit ပြုလုပ်ခြင်း)။ Not all operations are perfectly compensatable (e.g., email sent, SMS sent) ဖြစ်သောကြောင့် Saga design time တွင် compensability ကို consider ပြုရပြီး idempotency ကိုလည်း ensure ပြုရမည်ဖြစ်သည်။

### Circuit Breaker
Circuit Breaker သည် downstream service failure ကြောင့် cascading failure မဖြစ်ရန် ကာကွယ်သော pattern ဖြစ်သည်။ Electrical circuit breaker ကဲ့သို့ — failure threshold ကျော်လျှင် "open" ဖြစ်ကာ request ကို ချက်ချင်း fail fast ပြုသည်၊ ကာလတစ်ခုကြာပြီးနောက် "half-open" တွင် test request ပေးကြည့်ကာ service ပြန်ကောင်းလျှင် "closed" ဖြစ်သည်။ Resilience4j, Hystrix, Polly ကဲ့သို့ libraries များဖြင့် implement ပြုနိုင်သည်။

### Consistent Hashing
Consistent Hashing သည် distributed system တွင် nodes များပေါ်တွင် data ကို ဖြန့်ချိရာ hash ring (circular space) ကို အသုံးပြုသော technique ဖြစ်သည်။ Node တစ်ခု ထပ်ပေါင်းသည် (သို့မဟုတ်) ဖယ်ရှားသည့်အခါ K/N keys သာ relocate ဖြစ်ရသည် (N = node count, K = total keys)။ Load balancing, distributed caching (Redis Cluster, Memcached), database sharding တို့တွင် widely used ဖြစ်သည်။

### CQRS (Command Query Responsibility Segregation)
CQRS သည် write operations (Commands) နှင့် read operations (Queries) ကို သီးခြား models နှင့် infrastructure ဖြင့် ခွဲထုတ်သော architectural pattern ဖြစ်သည်။ Write side တွင် normalized relational DB (e.g., PostgreSQL) ကို သုံးနိုင်ပြီး read side တွင် denormalized, query-optimized store (e.g., Elasticsearch, Redis) ကို သီးခြားသုံးနိုင်သည်။ Event Sourcing နှင့် တွဲဖက်သုံးလေ့ရှိပြီး complex domain models တွင် scalability နှင့် performance တိုးတက်ကောင်းမွန်စေသည်။

---

## D

### DDD (Domain-Driven Design)
DDD (Domain-Driven Design) သည် Eric Evans မှ မိတ်ဆက်ခဲ့သော software design approach ဖြစ်ပြီး complex business domain ၏ logic ကို code ၏ core တွင် ထားကာ business experts နှင့် developers တို့ shared language (ubiquitous language) ဖြင့် ပူးပေါင်းဆောင်ရွက်ရသည်။ Bounded Context, Aggregate, Entity, Value Object, Domain Event, Repository ကဲ့သို့ building blocks များဖြင့် တည်ဆောက်သည်။ Microservices architecture ၏ service boundary ကို design ရာတွင် DDD သည် standard reference model ဖြစ်လာသည်။

### Dead Letter Queue (DLQ)
Dead Letter Queue (DLQ) သည် processing မအောင်မြင်သော messages (maximum retry ကျော်သော၊ expiry ကုန်သော၊ format မမှန်သော) ကို သိမ်းဆည်းသော special queue ဖြစ်သည်။ Main processing queue မှ error messages ကို သီးခြားထားကာ investigation, manual reprocessing, alerting အတွက် အသုံးပြုနိုင်သည်။ Kafka, SQS, RabbitMQ တို့တွင် DLQ ကို built-in support ဖြင့် configure ပြုနိုင်သည်။

---

## E

### Eventual Consistency
Eventual Consistency သည် distributed system တွင် new writes ပြီးနောက် data ကို reads ဖြင့် ချက်ချင်းမမြင်ရနိုင်ဘဲ — network partition မဖြစ်ဘဲ လုံလောက်သောအချိန်ကြာပြီးနောက် nodes အားလုံးသည် same value သို့ converge မည်ကို guarantee ပေးသော consistency model ဖြစ်သည်။ Strong consistency ထက် lower latency နှင့် higher availability ကို ရသော်လည်း stale reads ဖြစ်နိုင်သည်ကို accept ပြုရသည်။ Cassandra, DNS, Amazon S3 တို့တွင် eventual consistency ကို default ဖြစ်သည်။

### Event Sourcing
Event Sourcing သည် application state ကို current state value တစ်ခုတည်းသိမ်းဆည်းမည့်အစား state change ဖြစ်ရပ်တစ်ခုချင်းစီ (events) ကို append-only log ဖြစ်အောင် သိမ်းဆည်းသော pattern ဖြစ်သည်။ Current state ကို event log ကို replay ဖြင့် reconstruct ပြုနိုင်ပြီး time-travel debugging, audit log, CQRS integration ကဲ့သို့ benefits ပေးသည်။ EventStore, Kafka, PostgreSQL (append table) တို့ကို event store ဖြစ်ကာ implementation ပြုနိုင်သည်။

### FHIR (Fast Healthcare Interoperability Resources)
FHIR (Fast Healthcare Interoperability Resources) သည် HL7 organization မှ develop ပြုသော healthcare data exchange ၏ modern standard ဖြစ်ပြီး RESTful API ဖြင့် standardized JSON/XML format ဖြင့် clinical data ကို exchange ပြုနိုင်စေသည်။ Patient, Observation, Condition, MedicationRequest ကဲ့သို့ "Resources" ဖြင့် healthcare data ကို model လုပ်ပြီး LOINC, ICD-10, RxNorm ကဲ့သို့ standard code systems ဖြင့် interoperability ကို ensure ပြုသည်။ Hospitals, pharmacies, insurance companies, labs တို့အကြား seamless data sharing ဖြစ်စေရန် US 21st Century Cures Act ဖြင့် FHIR adoption ကို mandate ပြုထားသည်။

---

## F

### Fan-out
Fan-out သည် single event / message / write operation ကို multiple destinations (queues, subscribers, services) သို့ distribute ပြုသော pattern ဖြစ်သည်။ Push-based model တွင် (e.g., Twitter home timeline) user ၏ tweet တစ်ခု post ဖြစ်သည့်အခါ followers အားလုံး၏ timeline cache ထဲသို့ write ဖြင့် "fan out on write" ဟုဆိုသည်။ Kafka topic တစ်ခုမှ consumer groups များစွာ subscribe ပြုသည်ကိုလည်း fan-out ဟုဆိုနိုင်ပြီး notification, analytics, audit log processing ကဲ့သို့ use cases တွင် common ဖြစ်သည်။

---

## G

### Geohash
Geohash သည် geographic coordinates (latitude, longitude) ကို hierarchical string token (e.g., `w3gv2c`) ဖြင့် encode ပြုသော system ဖြစ်သည်။ String length ရှည်လာသည်နှင့်အမျှ precision မြင့်မားပြီး ပထဝီဝင် proximity ကို string prefix comparison ဖြင့် approximate ပြုနိုင်သောကြောင့် nearby location queries တွင် efficient ဖြစ်သည်။ Uber, Lyft, Yelp ကဲ့သို့ geospatial services များတွင် ride matching, nearby search, geofencing ရည်ရွယ်ချက်ဖြင့် အသုံးပြုသည်။

### GitOps
GitOps သည် infrastructure နှင့် application deployment ကို Git repository ကို "single source of truth" ဖြစ်အောင် ထားကာ reconciliation loop (e.g., ArgoCD, Flux) မှ desired state (Git) နှင့် actual state (cluster) တို့ကို continuously sync ပြုသော operational model ဖြစ်သည်။ Kubernetes environment တွင် widely adopted ဖြစ်ပြီး deployment history, rollback, audit trail ကို Git history မှတ်တမ်းဖြင့် ရနိုင်သည်။ Manual `kubectl apply` ကို replace ဖြင့် declarative, automated deployment pipeline ဖြစ်စေသည်။

### gRPC
gRPC သည် Google မှ ဖန်တီးသော high-performance, open-source RPC (Remote Procedure Call) framework ဖြစ်ပြီး HTTP/2 ကို transport layer ဖြစ်ကာ Protocol Buffers ကို serialization format ဖြစ်ကာ သုံးသည်။ Unary, server streaming, client streaming, bidirectional streaming ဆိုသော communication patterns လေးမျိုးကို support ပြုပြီး `.proto` file မှ code generation ဖြင့် type-safe client/server stubs ကို auto-generate ပြုနိုင်သည်။ Microservices ၏ internal service-to-service communication တွင် REST ထက် bandwidth နှင့် latency ကို improve ပြုနိုင်ရာ popular choice ဖြစ်သည်။

### HIPAA (Health Insurance Portability and Accountability Act)
HIPAA သည် United States ၏ federal law ဖြစ်ပြီး Protected Health Information (PHI) ၏ privacy နှင့် security ကို protect ပြုရန် healthcare providers, health plans, clearinghouses (covered entities) နှင့် ၎င်းတို့၏ business associates တို့ကို regulate ပြုသည်။ Technical safeguards (encryption at rest AES-256, in transit TLS 1.3), administrative safeguards (access control, workforce training), physical safeguards (facility access) ဟူသော requirements သုံးမျိုးကို comply ပြုရပြီး PHI access audit logs ကို 6+ years ထိန်းသိမ်းရမည်ဖြစ်သည်။ Data breach ဖြစ်ပါက 60 days အတွင်း HHS (Department of Health and Human Services) သို့ report ပြုရပြီး violation penalties သည် $100 to $1.9M per violation category per year ရှိနိုင်သည်။

---

## H

### Helm
Helm သည် Kubernetes ၏ package manager ဖြစ်ပြီး "charts" ဟုခေါ်သော pre-configured Kubernetes resource templates များကို packaging, versioning, sharing ပြုနိုင်သည်။ `helm install`, `helm upgrade`, `helm rollback` commands ဖြင့် application deployment ကို manage လုပ်နိုင်ပြီး values.yaml ဖြင့် environment-specific configuration ကို parameterize ပြုနိုင်သည်။ Complex Kubernetes deployments (databases, monitoring stacks, services) ၏ repeatability နှင့် consistency ကို ensure ပြုသည်။

### HyperLogLog
HyperLogLog သည် large datasets တွင် unique elements ၏ cardinality (approximate count of distinct items) ကို sub-linear memory (log log n space) ဖြင့် estimate ပြုသော probabilistic algorithm ဖြစ်သည်။ Standard error ≈ 0.81% ဖြင့် billion-scale unique counts ကို kilobytes memory သာဖြင့် estimate ပြုနိုင်သောကြောင့် exact count ကို exact memory မလိုသည့် use cases တွင် popular ဖြစ်သည်။ Redis ၏ `PFADD` / `PFCOUNT` commands မှတဆင့် daily active users, unique page views, unique search queries counting တို့တွင် widely used ဖြစ်သည်။

---

## I

### Idempotency
Idempotency ဆိုသည်မှာ operation တစ်ခုကို တစ်ကြိမ် (သို့မဟုတ်) ထပ်တလဲလဲ execute ပြုသော်လည်း outcome / side effect တူညီနေသော property ကိုဆိုသည်။ Distributed systems တွင် network retry ကြောင့် duplicate requests ဖြစ်တတ်သောကြောင့် payment API, order creation, database upsert ကဲ့သို့သော critical operations တွင် idempotency key (unique request ID) ဖြင့် duplicate ကို detect ပြီး ထပ်မလုပ်ဘဲ cached response ပြန်ပေးသည်။ REST တွင် GET, PUT, DELETE သည် idempotent ဖြစ်ပြီး POST သည် default ဖြင့် idempotent မဟုတ်ပါ။

---

## J

### JWT (JSON Web Token)
JWT (JSON Web Token) သည် parties နှစ်ဖက်ကြားတွင် claims ကို securely transmit ရန် compact, URL-safe token format ဖြစ်ပြီး Header.Payload.Signature ဟူ၍ dot-separated three parts ဖြင့် ဖွဲ့စည်းသည်။ Server-side session storage မလိုဘဲ stateless authentication ကို enable ပြုသောကြောင့် microservices inter-service authentication, mobile app authentication တို့တွင် popular ဖြစ်သည်။ Signature ကို server ၏ secret key (HMAC) သို့မဟုတ် private key (RSA/ECDSA) ဖြင့် verify ပြုနိုင်ပြီး token ၏ integrity ကို ensure ပြုသည်။

---

## K

### Kafka
Apache Kafka သည် distributed event streaming platform ဖြစ်ပြီး high-throughput, durable, fault-tolerant message log ကို topic-based publish-subscribe model ဖြင့် provide ပြုသည်။ Messages ကို consumer ack ပြီးနောက် ချက်ချင်းမဖျက်ဘဲ configurable retention period (default 7 days) ထိ disk တွင် သိမ်းဆည်းသောကြောင့် event replay capability ကို support ပြုသည်။ Event sourcing, log aggregation, real-time analytics pipeline, microservices decoupling ကဲ့သို့ use cases တွင် de facto standard ဖြစ်လာပြီး LinkedIn မှ တည်ဆောက်ကာ Apache Foundation ၏ open-source project ဖြစ်သည်။

### Kubernetes
Kubernetes (K8s) သည် container workloads ကို automate ဖြင့် deploy, scale, manage ပြုနိုင်သော open-source container orchestration platform ဖြစ်သည်။ Declarative configuration (YAML manifests) ဖြင့် desired state ကို သတ်မှတ်လျှင် control plane မှ actual state ကို desired state နှင့် reconcile ပြုမည်ကို continuously ဆောင်ရွက်သည်။ Self-healing (failed pods restart), auto-scaling (HPA/VPA), service discovery, rolling deployments ကဲ့သို့ features များဖြင့် microservices deployment ၏ operational complexity ကို significantly လျော့ချနိုင်သည်။

---

## M

### Microservice
Microservice သည် single business capability (e.g., Order, Payment, User) ကို independently deploy နိုင်သော small, autonomous service ဖြစ်ပြီး well-defined API ဖြင့် အခြား services နှင့် communicate ပြုသည်။ Monolith ကဲ့သို့ large single deployment unit မဟုတ်ဘဲ service တစ်ခုချင်းစီ သည် own database, own deployment pipeline, own technology stack ကို ရနိုင်ပြီး small team တစ်ခုဖြင့် manage ပြုနိုင်သည်။ Netflix, Amazon, Uber ကဲ့သို့ ကြီးမားသော tech companies များသည် scaling, deployment flexibility, team autonomy ကြောင့် monolith မှ microservices သို့ migrate ပြုခဲ့သည်။

### Monolith
Monolith သည် application ၏ modules အားလုံး (UI, business logic, data access) ကို single deployable unit ဖြစ်ကာ တည်ဆောက်သော traditional architectural style ဖြစ်သည်။ Simple deployment, easy debugging, good performance (no network hops) ကဲ့သို့ advantages ရှိသော်လည်း codebase growth နှင့်အမျှ deployments ပိုနှေး၊ scaling granular မဖြစ်နိုင်၊ teams coupling ဖြစ်ကာ development velocity ကျဆင်းသော disadvantages ရှိသည်။ Startup early stage တွင် monolith ဖြင့် စတင်ကာ complexity မြင့်လာသောအခါ "Strangler Fig Pattern" ဖြင့် gradual microservices migration ကို recommend ပြုလေ့ရှိသည်။

### mTLS (Mutual TLS)
mTLS (Mutual TLS) သည် standard TLS ၏ one-way authentication (server only proves identity) ကို extend ဖြင့် client နှင့် server နှစ်ဦးစလုံး X.509 certificates ဖြင့် identity prove ပြုရသော authentication mechanism ဖြစ်သည်။ Service mesh (Istio, Linkerd) environment တွင် microservices ကြားသော east-west traffic ကို mTLS ဖြင့် automatically encrypt ပြုကာ mutual authentication ပြုနိုင်ပြီး impersonation attacks ကို prevent ပြုနိုင်သည်။ Certificate rotation, certificate authority management ကဲ့သို့ operational complexity ရှိသော်လည်း zero-trust network security model တွင် essential component ဖြစ်သည်။

---

## P

### Protobuf (Protocol Buffers)
Protobuf (Protocol Buffers) သည် Google မှ develop ပြုထားသော language-neutral, platform-neutral binary serialization format ဖြစ်ပြီး `.proto` file ဖြင့် schema ကို define ပြီး code generator ဖြင့် multiple languages (Java, Go, Python, C++) ၏ serialization/deserialization code ကို auto-generate ပြုသည်။ JSON ထက် 3-10x smaller payload size နှင့် 20-100x faster serialization performance ပေးသောကြောင့် gRPC ၏ default serialization format ဖြစ်ပြီး high-performance inter-service communication တွင် widely used ဖြစ်သည်။ Schema evolution ကို field numbering system ဖြင့် support ပြုပြီး backward/forward compatibility ကို maintain ပြုနိုင်သော်လည်း human-readable မဟုတ်သောကြောင့် debugging တွင် tooling support လိုအပ်သည်။

### PCI-DSS (Payment Card Industry Data Security Standard)
PCI-DSS သည် credit/debit card data ကို handle ပြုသော organizations အားလုံး comply ပြုရသော security standard ဖြစ်ပြီး Visa, Mastercard, American Express တို့မှ ဖွဲ့စည်းသော PCI Security Standards Council မှ maintain ပြုသည်။ Cardholder data (PAN, CVV, expiry) ကို encrypt ပြုခြင်း, network segmentation, vulnerability scanning, access control, penetration testing ကဲ့သို့ requirements 12 ခု (6 categories) ကို satisfy ပြုရပြီး compliance level သည် annual transaction volume ပေါ်မူတည်သည်။ Payment microservices design တွင် PCI scope minimization (tokenization ဖြင့် real card numbers ကို service boundaries ပြင်ပမထုတ်ခြင်း) သည် critical strategy ဖြစ်ပြီး PCI-compliant payment gateway (Stripe, Adyen) integration ဖြင့် scope ကို significantly reduce ပြုနိုင်သည်။

---

## O

### OAuth2
OAuth2 သည် user credentials ကို third-party application နှင့် share မဖြစ်ဘဲ limited access ကို delegate ပြုနိုင်သော authorization framework ဖြစ်ပြီး "Login with Google/Facebook" ကဲ့သို့ social login scenarios တွင် foundation ဖြစ်သည်။ Authorization Code Flow, Client Credentials Flow, Implicit Flow, Device Flow ကဲ့သို့ grant types များဖြင့် different scenarios တွင် secure token exchange ကို enable ပြုသည်။ OpenID Connect (OIDC) သည် OAuth2 ၏ authorization layer ပေါ်တွင် authentication layer ကို ထပ်ဆောင်းထားကာ modern application authentication ၏ industry standard ဖြစ်သည်။

### Outbox Pattern
Outbox Pattern သည် database transaction နှင့် event publishing ကို atomically ဆောင်ရွက်ရန် — DB transaction အတွင်း application state table နှင့် outbox table (pending events) နှစ်ခုလုံးကို single transaction ဖြင့် write ပြုသော pattern ဖြစ်သည်။ Separate message relay process (e.g., Debezium CDC, polling publisher) မှ outbox table ကို read ပြီး message broker သို့ reliably publish ပြုကာ published records ကို delete (သို့မဟုတ်) mark ပြုသည်။ "Dual write" ပြဿနာ (DB write success ဖြစ်ပြီး Kafka publish fail ဖြစ်ကာ inconsistency ဖြစ်နိုင်) ကို ဖြေရှင်းသောကြောင့် event-driven microservices တွင် highly recommended pattern ဖြစ်သည်။

---

## R

### Rate Limiter
Rate Limiter သည် user / IP / API key တစ်ခုမှ time window တစ်ခုအတွင်း request ပမာဏကို ကန့်သတ်သော mechanism ဖြစ်ပြီး API abuse, DDoS attacks, resource starvation ကို prevent ပြုသည်။ Token Bucket, Leaky Bucket, Fixed Window, Sliding Window, Sliding Window Log ကဲ့သို့ algorithms ဖြင့် implement ပြုနိုင်ပြီး Redis ကို counter storage ဖြစ်ကာ distributed rate limiting ကို enable ပြုလေ့ရှိသည်။ HTTP 429 (Too Many Requests) response code နှင့် `Retry-After` header ဖြင့် client ကို throttle ပြုသည်ကို inform ပြုသည်။

### RPO (Recovery Point Objective)
RPO (Recovery Point Objective) သည် disaster / failure ဖြစ်ပြီးနောက် business ၏ acceptable data loss ပမာဏကို time unit ဖြင့် ဖော်ပြသော metric ဖြစ်သည်။ RPO = 1 hour ဆိုလျှင် — last backup မှ ယခုအချိန်ထိ maximum 1 hour ၏ data loss ကို accept ပြုနိုင်ကာ backup frequency ကို at least every 1 hour ပြုလုပ်ရသည်ကို ဆိုလိုသည်။ Business impact analysis ဖြင့် သတ်မှတ်ကာ backup strategy, replication mechanism, storage solution ကို drive ပြုသော business requirement ဖြစ်သည်။

### RTO (Recovery Time Objective)
RTO (Recovery Time Objective) သည် disaster ဖြစ်ပြီးနောက် system ကို acceptable operational state သို့ restore ပြုနိုင်ရမည့် maximum time ကို ဖော်ပြသော metric ဖြစ်သည်။ RTO = 4 hours ဆိုလျှင် — failure မှ 4 နာရီ အတွင်း service ပြန်ကောင်းရမည်ဟု ဆိုလိုပြီး infrastructure automation, failover mechanism, on-call procedures တို့ကို ဒီ target နှင့် align ဖြစ်ရမည်သည်။ RPO နှင့် RTO တို့သည် Disaster Recovery plan ၏ core metrics ဖြစ်ပြီး SLA ၏ downtime commitment ကဲ့သို့ business continuity requirements မှ derive ဖြစ်ကာ အများသောအားဖြင့် ငွေကြေးဆင်းသည်နှင့်အမျှ target တင်းကျပ်လာသည်။

---

## S

### Schema Registry
Schema Registry သည် event/message schemas (Avro, Protobuf, JSON Schema) ကို centrally store, version, validate ပြုသော service ဖြစ်ပြီး Kafka ecosystem တွင် Confluent Schema Registry သည် de facto standard ဖြစ်သည်။ Producers သည် message publish ပြုသောအခါ schema ကို registry ထဲတွင် register ပြုပြီး consumers သည် schema ID ဖြင့် correct deserialization schema ကို fetch ပြုနိုင်သည်။ Schema compatibility checks (backward, forward, full) ကို enforce ပြုသောကြောင့် breaking changes ကို production deploy မတိုင်မီ detect ပြုနိုင်ပြီး event-driven microservices ၏ schema evolution safety net ဖြစ်သည်။

### Saga
Saga သည် multiple microservices ၏ local transactions များကို sequence ဖြင့် ချိတ်ဆက်ကာ distributed transaction ကဲ့သို့ long-running business process ကို achieve ပြုသော pattern ဖြစ်သည်။ ACID 2-phase commit မသုံးဘဲ — failure ဖြစ်ပါက compensating transactions ဖြင့် already-completed steps ကို undo ဖြင့် data consistency ကို maintain ပြုသည်။ Choreography (event-driven, decentralized) နှင့် Orchestration (central coordinator) ဆိုသော approaches နှစ်မျိုးဖြင့် implement ပြုနိုင်ပြီး e-commerce checkout, bank transfer, booking system ကဲ့သို့ multi-step business processes တွင် applicable ဖြစ်သည်။

### Service Mesh
Service Mesh သည် microservices ၏ inter-service communication ကို application code မပြောင်းဘဲ infrastructure level တွင် handle ပြုသော dedicated infrastructure layer ဖြစ်ပြီး sidecar proxy pattern (e.g., Envoy) ဖြင့် implement ပြုသည်။ mTLS encryption, load balancing, circuit breaking, retries, distributed tracing, traffic management (canary, A/B testing) ကဲ့သို့ cross-cutting concerns များကို services မှ offload ပြုကာ central control plane (Istio, Linkerd) မှ observe ပြုနိုင်သည်။ Large Kubernetes deployments တွင် observability နှင့် security ကို significantly improve ပြုသော်လည်း additional infrastructure complexity နှင့် latency overhead ကို bring ပြုသောကြောင့် genuinely complex environments တွင်သာ justify ဖြစ်သည်။

### Sharding
Sharding သည် large database ကို smaller, independently manageable pieces (shards) ဖြင့် horizontal partition ပြုသော technique ဖြစ်ပြီး shard တစ်ခုချင်းစီသည် data ၏ subset ကိုသာ ဆောင်ရွက်သည်။ Shard key (e.g., user ID, geographic region) ကို သတ်မှတ်ကာ hash-based, range-based, directory-based sharding strategies ဖြင့် data routing ကို determine ပြုသည်။ Write/read throughput ကို linearly scale ပြုနိုင်ပြီး single node ၏ hardware limit ကို overcome ပြုနိုင်သော်လည်း cross-shard queries, transactions, re-sharding complexity ကဲ့သို့ operational challenges ကို introduce ပြုသောကြောင့် design phase တွင် careful planning လိုအပ်သည်။

### SSE (Server-Sent Events)
SSE (Server-Sent Events) သည် server မှ client သို့ one-directional real-time data push ပြုနိုင်သော HTTP-based protocol ဖြစ်ပြီး `text/event-stream` content type ဖြင့် long-lived HTTP connection ပေါ်တွင် events ကို stream ပြုသည်။ WebSocket နှင့် မတူဘဲ unidirectional (server-to-client only) ဖြစ်သော်လည်း standard HTTP infrastructure (proxies, load balancers) ဖြင့် ကောင်းစွာ compatible ဖြစ်ပြီး automatic reconnection, event ID tracking ကဲ့သို့ built-in features ပါရှိသည်။ Real-time notifications, live dashboard updates, stock price feeds ကဲ့သို့ server-push-only scenarios တွင် WebSocket ထက် simpler implementation ဖြစ်ပြီး sufficient ဖြစ်သော option ဖြစ်သည်။

### SLA (Service Level Agreement)
SLA (Service Level Agreement) သည် service provider နှင့် customer ကြားတွင် service quality metrics (uptime, response time, error rate) ကို formally commit ပြုသော contractual agreement ဖြစ်သည်။ Breach ဖြစ်ပါက financial penalties (service credits) ကဲ့သို့ consequences ကို define ပြုပြီး external-facing customers နှင့် agreements တွင် legally binding ဖြစ်သည်။ SLO (Service Level Objective) သည် internal target ဖြစ်ပြီး SLA သည် ထို SLO ကို conservative margin ဖြင့် customer သို့ commit ပြုသော external commitment ဖြစ်သောကြောင့် SLA ≤ SLO ဖြစ်ကြောင်း engineering teams မှ remember ပြုရမည်ဖြစ်သည်။

### SLO (Service Level Objective)
SLO (Service Level Objective) သည် service ၏ reliability target ကို specific metric (e.g., availability 99.9%, P99 latency ≤ 200ms) ဖြင့် အတိအကျ သတ်မှတ်သော internal goal ဖြစ်သည်။ SLI (Service Level Indicator) ကို measure ဖြင့် SLO ၏ current achievement ကို assess ပြုပြီး error budget (100% - SLO%) ကို feature development vs reliability work ပြုလုပ်ချိန် balance ရာတွင် guide ဖြစ်ကာ Google SRE ၏ core practice ဖြစ်သည်။ SLO ကို product, business, reliability တော်တော်များများ joint ဖြင့် agree ပြုကာ ops team ၏ pager alerting threshold ကိုသာ မဟုတ်ဘဲ engineering culture ကို drive ပြုသော metric ဖြစ်သည်ကို recognize ပြုရသည်။

---

## T

### Two-Phase Commit (2PC)
Two-Phase Commit (2PC) သည် distributed transaction တွင် multiple participants (databases, services) အားလုံး commit ပြုမည် (သို့မဟုတ်) အားလုံး abort ပြုမည်ဟု guarantee ပေးသော atomic commit protocol ဖြစ်သည်။ Phase 1 (Prepare) တွင် coordinator မှ participants အားလုံးကို "commit ပြုနိုင်မည်လား" ဟု vote request ပေးပြီး Phase 2 (Commit/Abort) တွင် participants အားလုံး "yes" vote ပေးမှသာ commit ပြု ပြီး တစ်ခုတည်းမှ "no" ဆိုလျှင် abort ပြုသည်။ Microservices architecture တွင် 2PC သည် blocking nature (coordinator fail ဖြစ်ပါက participants lock ခံရသည်), latency overhead, tight coupling ကြောင့် recommend မပြုဘဲ Saga pattern (compensating transactions) ကို alternative ဖြစ်ကာ widely prefer ပြုကြသည်။

---

## W

### WebSocket
WebSocket သည် HTTP upgrade handshake ပြီးနောက် client နှင့် server ကြား bidirectional, full-duplex communication channel ကို persistent TCP connection ပေါ်တွင် establish ပြုသော protocol (RFC 6455) ဖြစ်သည်။ HTTP request-response model မဟုတ်ဘဲ both sides မှ ချက်ချင်း message push ပြုနိုင်သောကြောင့် real-time chat, multiplayer games, live collaboration, live trading platforms ကဲ့သို့ low-latency bidirectional communication လိုအပ်သော applications တွင် essential ဖြစ်သည်။ Load balancing တွင် sticky sessions လိုအပ်ခြင်း, connection state management complexity, horizontal scaling ခက်ခဲခြင်း ကဲ့သို့ operational challenges ရှိသောကြောင့် truly bidirectional communication မလိုအပ်ပါက SSE (Server-Sent Events) ကို simpler alternative ဖြစ်ကာ consider ပြုသင့်သည်။

---

## Z

### Zero Trust
Zero Trust သည် "network ၏ inside ဖြစ်သောကြောင့် trust ပြုသည်" ဟူသော traditional perimeter security model ကို reject ဖြင့် — မည်သည့် request မဆိုကို (inside network မှ ဖြစ်သော်လည်း) "never trust, always verify" ဟူ၍ treat ပြုသော security philosophy ဖြစ်သည်။ Microservices environment တွင် service-to-service communication ကို mTLS, service identity (SPIFFE/SPIRE), API authorization ဖြင့် authenticate ပြုကာ network segmentation, principle of least privilege, continuous monitoring တို့ကို enforce ပြုသည်။ Traditional VPN-based security ထက် modern cloud-native, remote-work environments တွင် more resilient ဖြစ်ပြီး NIST SP 800-207 သည် Zero Trust Architecture ၏ standard reference ဖြစ်သည်။

---

## မှတ်ချက်

ဤ glossary တွင် မပါဝင်သေးသော terms များကို ညာဘက် QR code (ebook version) သို့မဟုတ် companion website မှ ရှာဖွေနိုင်ပါသည်။ Technical terms များသည် rapidly evolving ဖြစ်သောကြောင့် ဤစာအုပ်ပုံနှိပ်ကာလ (2026) မှ context ဖြင့် ဖတ်ရှုပါ။
