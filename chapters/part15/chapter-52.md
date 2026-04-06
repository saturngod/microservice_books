# အခန်း ၅၂: Microservices Interview Playbook

## နိဒါန်း

System Design Interview သည် senior software engineer position များအတွက် most important screening ဖြစ်သည်။ FAANG (Facebook/Meta, Amazon, Apple, Netflix, Google) နှင့် tier-1 tech companies တို့တွင် system design round ကို pass မဖြစ်ပါက ဖိတ်ကြားခံရနိုင်မည် မဟုတ်ပါ။

ဤ playbook ကို memorize ပြုမည်မဟုတ်ဘဲ framework ကို internalize ပြုကာ any question ကို structured approach ဖြင့် answer ပေးနိုင်ရမည်ဖြစ်သည်။ Interviewer တို့သည် specific solution ကို လိုချင်ခြင်းမဟုတ်ဘဲ candidate ၏ thinking process, trade-off awareness, communication skill တို့ကို ကြည့်ကြသည်ဆိုသည်ကို နားလည်ရမည်ဖြစ်သည်။

---

## ၅၂.၁ How to Approach Any System Design Question (Framework)

### The RESHADE Framework

```
R - Requirements Clarification  (5 minutes)
E - Estimation                   (5 minutes)
S - System Interface Design      (5 minutes)
H - High-Level Architecture      (10 minutes)
A - API Design                   (5 minutes)
D - Data Model Design            (5 minutes)
E - Elaborate Deep Dives         (15 minutes)
```

### Step 1: Requirements Clarification (5 min)

```
Questions to ask:
"Before I start designing, I'd like to clarify requirements."

Functional:
□ Who are the users? (riders, drivers, admins)
□ What are the core features? (Must-have vs nice-to-have)
□ What platforms? (iOS, Android, Web)
□ What's out of scope? (What NOT to design)

Non-Functional:
□ Scale: How many users? DAU? QPS?
□ Latency: Real-time or eventual?
□ Consistency: Strong or eventual?
□ Availability: 99.9% or 99.99%?
□ Geographic: Single region or global?
□ Data retention: How long?

// MISTAKE TO AVOID: Immediately starting to draw
// without understanding requirements
```

### Step 2: Estimation (5 min)

```
ခန့်မှန်းချက်ကို always show your math

Traffic:
DAU: 100 million users
Writes per user per day: 2
Reads per user per day: 10

Write QPS = 100M × 2 / 86,400 ≈ 2,300 QPS
Read QPS  = 100M × 10 / 86,400 ≈ 11,600 QPS
Peak (3x) = Write: ~7K, Read: ~35K

Storage:
Per record: 1 KB
Daily new records: 2 × 100M = 200M
Daily storage: 200M × 1 KB = 200 GB/day
5 years storage: 200 GB × 365 × 5 = 365 TB

Bandwidth:
Read bandwidth = 35K QPS × 1 KB = 35 MB/s
Write bandwidth = 7K QPS × 1 KB = 7 MB/s
```

### Step 3: High-Level Architecture (10 min)

```
Always draw these components:
1. Client (Mobile/Web)
2. Load Balancer / API Gateway
3. Core Services (2-4 major ones)
4. Cache Layer (Redis/Memcached)
5. Database(s)
6. Message Queue (Kafka) if needed
7. CDN if media/static assets

// Speak while drawing:
"I'll start with a client that connects through
an API Gateway, which routes to our core services..."
```

### Step 4: Deep Dive (15 min)

```
Interviewer-guided or self-selected critical components:

Common deep dives:
- Database choice & schema
- Caching strategy
- Handling failures
- Scaling bottlenecks
- Consistency model
- Security considerations
```

---

## ၅၂.၂ Estimation Cheat Sheet

### QPS Reference Numbers

```
+------------------------------------------+
|  Service           | QPS Range           |
+------------------------------------------+
| Twitter-scale      | 600K read, 6K write |
| Instagram-scale    | 500K read, 50K write|
| Uber (peak)        | 100K ride requests  |
| Google Search      | 40,000 QPS global   |
| YouTube            | 1M+ video requests  |
| WhatsApp messages  | 1M+ msg/sec         |
+------------------------------------------+

// Rule of thumb:
// 1 QPS = 86,400 requests/day
// 1M/day ≈ 12 QPS
// 1B/day ≈ 12,000 QPS (12K QPS)
// 1M users × 10 req/day = 100M/day ≈ 1,200 QPS
```

### Storage Reference Numbers

```
+------------------------------------------+
|  Data Type     | Typical Size            |
+------------------------------------------+
| User record    | 1 KB                    |
| Tweet/post     | 500 bytes               |
| Photo metadata | 300 bytes               |
| Photo (mobile) | 500 KB - 2 MB           |
| Video (1 min)  | 50-100 MB               |
| Audio (1 min)  | 1-5 MB                  |
| Location point | 50 bytes                |
+------------------------------------------+

Storage shortcuts:
1 KB = 10^3 bytes
1 MB = 10^6 bytes
1 GB = 10^9 bytes
1 TB = 10^12 bytes
1 PB = 10^15 bytes

// Example: Instagram
// 500M active users upload 100M photos/day
// Average photo: 200 KB
// Daily storage: 100M × 200KB = 20 TB/day
// Yearly: 20TB × 365 = 7.3 PB/year
```

### Latency Reference Numbers (Must Memorize)

```
+------------------------------------------+
|  Operation              | Latency        |
+------------------------------------------+
| L1 Cache                | 1 ns           |
| L2 Cache                | 10 ns          |
| L3 Cache / RAM          | 100 ns         |
| SSD random read         | 100 μs (0.1ms) |
| HDD seek                | 10 ms          |
| Redis GET               | < 1 ms         |
| Intra-datacenter (LAN)  | 0.5 ms         |
| Same region API call    | 5 ms           |
| Cross-continent         | 100-200 ms     |
| DNS lookup              | 20-120 ms      |
| CDN edge serve          | 5-50 ms        |
+------------------------------------------+

Memory shortcuts:
"L1 to RAM = 1, 10, 100 ns (10x each)"
"SSD = 100μs, Cross-continent = 100ms (×1000)"
```

### Bandwidth Reference Numbers

```
Single server capacity:
- CPU: ~16 cores standard
- RAM: 128-256 GB standard
- Network: 10 Gbps = 1.25 GB/s
- Disk I/O (SSD): ~1 GB/s read
- PostgreSQL: ~5,000-10,000 QPS per instance
- Redis: ~100,000 QPS per instance
- Kafka partition: ~10 MB/s throughput
```

---

## ၅၂.၃ Communication Pattern Decision Guide

```
Decision Tree:

Q: Does the sender need an immediate response?
|
+-- YES → Synchronous
|          |
|          +-- Simple request-response? → REST (HTTP)
|          |
|          +-- High performance, streaming? → gRPC
|          |
|          +-- Flexible querying? → GraphQL
|          |
|          +-- Real-time bidirectional? → WebSocket
|
+-- NO → Asynchronous
           |
           +-- Simple queue, single consumer? → SQS / RabbitMQ
           |
           +-- High throughput, multiple consumers,
           |   replay needed? → Kafka
           |
           +-- Fan-out to many services? → Pub/Sub (SNS/Kafka topics)
           |
           +-- Scheduled/delayed? → Job Queue (Celery, Temporal)
```

### When to Use What

| Pattern | Use Case | Avoid When |
|---------|----------|------------|
| REST | CRUD operations, public APIs | Streaming, real-time |
| gRPC | Inter-service, low latency | Public/browser clients |
| GraphQL | Flexible client queries | Simple, fixed responses |
| WebSocket | Real-time: chat, live updates | Infrequent updates |
| Kafka | Event streaming, high throughput | Simple task queues |
| SQS/RabbitMQ | Task queues, work distribution | Event replay needed |

---

## ၅၂.၄ Trade-off Templates (CAP, PACELC, Sync vs Async)

### CAP Theorem Application

```
CAP = Consistency, Availability, Partition Tolerance
(P always happens, so choose C or A)

CP Systems (Consistency over Availability):
- Banking transactions → "I'd rather show error than show wrong balance"
- Inventory management → "Overselling is worse than downtime"
- Leader election

AP Systems (Availability over Consistency):
- Social media likes/counts → "Slightly stale count is fine"
- DNS → "Stale cache better than lookup failure"
- Shopping cart → "Can resolve conflicts at checkout"
- User feeds

// Interview answer template:
"For [use case], I'd choose [CP/AP] because
[business reason]. The trade-off is [what we sacrifice],
which is acceptable because [justification]."
```

### PACELC Theorem

```
PACELC extends CAP:
P → choose A or C (during partition)
E → choose L (Latency) or C (Consistency) when no partition

Database PACELC profiles:
- DynamoDB:  PA/EL-leaning in many distributed patterns; consistency mode ရွေးနိုင်
- Cassandra: PA/EL (same)
- MongoDB:   deployment-dependent; read/write concerns နှင့် topology ပေါ်မူတည်
- PostgreSQL: PC/EC (strong consistency, higher latency)
- Spanner:   PC/EC (global strong consistency, higher latency)

// Example answer:
"For our ride location tracking, I'd use DynamoDB
(PA/EL) because we can tolerate slightly stale location
data (seconds) for high availability. For payment records,
I'd use PostgreSQL (PC/EC) because we need strong
consistency to prevent double charges."
```

### Sync vs Async Decision Template

```
Choose Synchronous when:
✓ Caller needs immediate response to proceed
✓ Operation is fast (< 100ms)
✓ Strong consistency required
✓ Error handling must be immediate
Example: Drug interaction check, Payment authorization

Choose Asynchronous when:
✓ Caller doesn't need immediate result
✓ Operation can take time (> 100ms)
✓ High throughput needed
✓ Service decoupling wanted
✓ Operations can be retried independently
Example: Email sending, Report generation, Analytics events
```

---

## ၅၂.၅ Common Follow-Up Questions & Model Answers

### "How would you handle failures?"

```
Structure your answer:

1. Detection:
   "I'd use health checks (HTTP /health endpoint)
   and circuit breakers (Hystrix/Resilience4j pattern)"

2. Prevention:
   "Timeouts on all external calls (default: 3-5 seconds),
   rate limiting to prevent cascade overload"

3. Recovery:
   "Retry with exponential backoff + jitter for transient failures.
   Dead letter queues for persistent failures needing manual review."

4. Graceful degradation:
   "If Recommendation Service is down, show popular items instead.
   If Search is slow, show cached results."

5. Monitoring:
   "Alert on error rate > 1%, latency P99 > 500ms"
```

### "How would you scale this system?"

```
Tier-by-tier answer:

Horizontal scaling:
"Add more service instances behind load balancer.
Stateless services scale linearly."

Caching:
"Add Redis cache in front of database.
Target 95%+ cache hit rate for read-heavy workloads."

Database:
"Read replicas for read-heavy (80:20 read:write).
Sharding for write-heavy (by user_id hash)."

Async offloading:
"Move non-critical sync operations to async via Kafka."

Geographic:
"Multi-region deployment with edge caching (CDN)
for static content."
```

### "Why not just use a monolith?"

```
Good answer template:

"A monolith may actually be the right choice initially.
The benefits of microservices come with real costs:
operational complexity, distributed system challenges,
network overhead, and team coordination overhead.

I'd recommend starting with a well-structured modular
monolith and extracting services when:
1. A specific component needs independent scaling
2. Team size requires parallel development
3. Different components need different tech stacks
4. Release frequency differs significantly between components

For [current scenario], given [team size/scale], I'd
[recommend monolith / justify microservices] because [reason]."
```

### "How do you ensure data consistency?"

```
Framework answer:

1. Define consistency requirement:
   "Financial data → strong consistency (ACID)
    Social feed → eventual consistency acceptable"

2. Within-service consistency:
   "ACID transactions within a single service's database"

3. Cross-service consistency:
   "Saga pattern (choreography/orchestration) for distributed transactions.
    Avoid distributed transactions (2PC) in microservices."

4. Event-driven eventual consistency:
   "Publish events, consumers update their local state.
    Compensating transactions for rollback."

5. Idempotency:
   "All write operations must be idempotent to handle retries safely."
```

---

## ၅၂.၆ Top 10 Design Mistakes Interviewers Look For

### Mistake #1: Not Asking Clarifying Questions

```
BAD: Start drawing immediately
GOOD: "Before I start, I have a few clarifying questions.
       How many users are we expecting? Is this global
       or single region? What's the expected read/write ratio?"

Why it matters: Interviewer wants to see if you think
               about requirements before jumping to solutions
```

### Mistake #2: Ignoring Non-Functional Requirements

```
BAD: Only designing for functionality
GOOD: "For this design, key NFRs are:
       99.99% availability → requires redundancy at each layer
       < 100ms P99 latency → drives our caching strategy
       1B+ users → requires horizontal scaling"
```

### Mistake #3: Single Points of Failure

```
BAD Design:
  Load Balancer → Single App Server → Single Database

GOOD Design:
  Load Balancer (HA pair) → App Server Cluster (3+ instances)
  → Primary DB + Read Replicas + Automated failover

// Always ask: "What happens if this component fails?"
```

### Mistake #4: No Caching Strategy

```
BAD: Database for every request
GOOD: "I'd add Redis as a caching layer with:
       - Cache-aside for user profiles (15-minute TTL)
       - Write-through for critical data
       - Cache-around-key pattern for hot data
       Targeting 90%+ cache hit rate to reduce DB load"
```

### Mistake #5: Not Considering Database Choice

```
BAD: "I'll use a database"
GOOD: "For user data (structured, transactional) → PostgreSQL
       For session data (key-value, TTL) → Redis
       For time-series metrics → InfluxDB or TimescaleDB
       For document storage (flexible schema) → MongoDB
       For global scale with consistency → DynamoDB or Spanner"

// Justify your database choice with data access patterns
```

### Mistake #6: Ignoring Network Failure

```
BAD: Service calls without error handling
GOOD: "All inter-service calls will have:
       - Timeout: 3s default, 10s for complex operations
       - Retry: 3 attempts with exponential backoff
       - Circuit breaker: Open after 50% failure rate
       - Fallback: Default response or cached response"
```

### Mistake #7: Not Thinking About Hot Spots

```
BAD: "Shard by user_id" (without thinking about distribution)
Problem: Celebrity account (Justin Bieber problem)
         One shard gets 90% of traffic

GOOD: "For social media likes on popular posts, I'd use
       a write-through cache to absorb spike.
       For famous users, consider dedicated shard or
       in-memory aggregation before DB flush."
```

### Mistake #8: Over-Engineering

```
BAD: Proposing Kafka + 10 microservices for a simple CRUD app
GOOD: "For this scale (10K DAU), I'd start with a
       monolith + PostgreSQL + Redis cache.
       This handles 10K DAU easily and we can evolve
       to microservices if we reach 10M DAU."

Rule: Complexity must be justified by real requirements
```

### Mistake #9: No Monitoring/Observability

```
Every design should mention:
✓ Metrics: "Prometheus + Grafana for system metrics"
✓ Logging: "Structured logs → Elasticsearch → Kibana"
✓ Tracing: "Distributed tracing with Jaeger/Zipkin"
✓ Alerting: "PagerDuty alerts for P99 latency > 500ms"
✓ Dashboards: "SLO dashboards for error rate < 0.1%"
```

### Mistake #10: Not Discussing Security

```
Even if not asked, briefly mention:

Authentication: "JWT tokens, OAuth 2.0 for third-party"
Authorization: "RBAC, check permissions at API Gateway"
Encryption: "TLS for all external traffic, mTLS for inter-service"
Secrets: "AWS Secrets Manager or Vault for credentials"
Rate Limiting: "100 req/min per user by default"
Data: "Encrypt PII at rest (AES-256)"
```

---

## Interview Scoring Rubric

```
What interviewers evaluate:

1. Problem Decomposition (25%)
   - Correctly identifying components
   - Appropriate service boundaries
   - Clear API contracts

2. Technical Depth (25%)
   - Correct data structures
   - Algorithm choices
   - Database selection reasoning

3. Scale & Performance (20%)
   - Bottleneck identification
   - Appropriate caching
   - Horizontal scaling strategy

4. Reliability & Fault Tolerance (15%)
   - Failure scenarios considered
   - Recovery strategies
   - Monitoring approach

5. Communication (15%)
   - Clear explanation of decisions
   - Trade-off articulation
   - Responding to hints/feedback

// Scoring:
// Strong Hire:  All 5 areas addressed thoroughly
// Hire:         4/5 areas covered well
// No Hire:      Missing critical areas or fundamentally wrong approach
```

---

## Quick Reference Card

```
+------------------------------------------------------------------+
|                SYSTEM DESIGN QUICK REFERENCE                     |
+------------------------------------------------------------------+
| Metric          | Rule of Thumb                                  |
|-----------------|------------------------------------------------|
| 1M DAU          | ~12 QPS average, ~36 QPS peak (3x)             |
| 1B DAU          | ~12K QPS avg, ~36K QPS peak                    |
| 1 photo/user/day| ~1.2 GB/day per 10K users (avg 120KB/photo)    |
|                 |                                                |
| Cache target    | 95%+ hit rate for read-heavy systems           |
| DB replication  | 1 primary + 2 replicas minimum                 |
| Sharding        | Consider when DB > 1TB or > 10K QPS writes     |
|                 |                                                |
| Redis           | 100K+ QPS, sub-millisecond                    |
| PostgreSQL      | 5-10K QPS per instance                         |
| Kafka partition | 10MB/s throughput per partition                |
|                 |                                                |
| Synchronous     | < 100ms operations, need immediate response   |
| Asynchronous    | > 100ms, fire-and-forget, high throughput      |
|                 |                                                |
| Strong consistency | Payment, inventory, medical records         |
| Eventual        | Social feeds, likes, recommendations           |
+------------------------------------------------------------------+
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **RESHADE Framework:** Requirements → Estimation → System Interface → High-Level → API Design → Data Model → Elaborate. ဤ framework ကို အလွတ်ကျက်ပြီး interview room တွင် think aloud ဖြင့် apply ပြုနိုင်ရမည်ဖြစ်သည်။

2. **Estimation Matters:** Math ကို show ပြုကာ ခန့်မှန်းချက်ဆွဲရမည်ဖြစ်သည်။ ဂဏန်းများ exact မဖြစ်ရ မဟုတ်ဘဲ order of magnitude သည် မှန်ကန်ရမည်ဖြစ်သည်။ Estimation process ကို see ရသည်ကိုသာ interviewer သည် care ပြုသည်ဖြစ်သည်။

3. **Trade-off Articulation:** "I chose X over Y because..." ဟူသော pattern ဖြင့် decision တိုင်းကို justify ပြုရမည်ဖြစ်သည်။ Trade-offs မသိဘဲ solution တင်ပြသည်ကို interviewer တို့သည် red flag အဖြစ် ရှုမြင်ကြသည်ဖြစ်သည်။

4. **Failure is Part of Design:** Happy path ကိုသာ design မဟုတ်ဘဲ failure scenarios, recovery strategies, monitoring တို့ပါ ထည့်သင်းသော engineer သည် production experience ရှိသည်ဟု interviewer မြင်သည်ဖြစ်သည်။

5. **Communicate Constantly:** Silent ဖြင့် drawing မဟုတ်ဘဲ think aloud ပြုနေရမည်ဖြစ်သည်။ "I'm considering X because..., but Y might be better if..." ဟူသော pattern သည် technical communication skill ကို demonstrate ပြုသည်ဖြစ်သည်။

6. **Top 10 Mistakes Avoidance:** Single points of failure, no caching, no error handling, over-engineering — ဤ mistakes တို့ကို avoid ပြုနိုင်ရုံမျှဖြင့် interview performance ကို significantly မြင့်တင်နိုင်သည်ဖြစ်သည်။

7. **Context-Aware Design:** Netflix scale solution ကို startup scale problem သို့ apply ပြုခြင်းသည် over-engineering anti-pattern ဖြစ်သည်ဆိုသည်ကို recognize ပြုနိုင်ရမည်ဖြစ်ပြီး requirements နှင့် proportional solution ကိုသာ propose ပြုရမည်ဖြစ်သည်။
