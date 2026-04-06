# အခန်း ၅၁: Microservices Anti-Patterns

## နိဒါန်း

Microservices architecture ကို adopt ပြုရာတွင် organizations များသည် common pitfalls တစ်ချို့ထဲသို့ ကျရောက်ကြသည်။ ဤ anti-patterns များကို early ဖော်ထုတ်ကာ ကရင်ဆောင်နိုင်ခဲ့လျှင် ကုန်ကျစရိတ်ကြီးမားသော refactoring effort ကို ကာကွယ်နိုင်သည်။ 

ဤအခန်းတွင် real-world projects တွင် မကြာမကြာ ကြုံတွေ့ရသော anti-patterns ၇ ခုကို symptoms, root causes, solutions နှင့်တကွ လေ့လာသွားမည်ဖြစ်သည်။ Anti-pattern တစ်ခုချင်းစီကို ဖော်ထုတ်တတ်ခြင်းသည် senior engineer တစ်ဦးအဖြစ် distinguish ဖြစ်နိုင်သော capability တစ်ခုဖြစ်သည်။

---

## ၅၁.၁ Distributed Monolith

### ဘာဆိုသည်

"Microservices" ဟု ခေါ်သော်လည်း valud ကို standalone deploy မပြုနိုင်ဘဲ တစ်ချိန်တည်းမှာ deploy ပြုလုပ်ရသော system ဖြစ်သည်။ Physically distributed ဖြစ်သော်လည်း logically tightly coupled ဖြစ်နေသည်။

### Symptoms

```
Red Flags:
✗ Service A deploy ပြုလျှင် Service B, C, D ပါ တပြိုင်နက် deploy ပြုရသည်
✗ Integration tests တွင် 10+ services ကို run ရသည်
✗ Service တစ်ခု fail ဖြစ်သောအခါ cascading failures ဖြစ်သည်
✗ Schema change တစ်ခု = multiple services update ပြုလုပ်ရသည်
✗ Deployment windows ကြာရှည်သည် (hours, not minutes)
✗ One big shared library ကို services အားလုံး depend ပြုသည်
```

### Root Causes

```
1. Wrong service boundaries (Incorrect domain decomposition)
   - Data model ပေါ်မူတည်ကာ services ခွဲ (ဒေတာ layer splitting)
   - Technical layer ပေါ်မူတည်ကာ ခွဲ (Frontend/Backend/Database)
   - Domain boundaries ကို မသတ်မှတ်ဘဲ ခွဲ

2. Shared databases (shared mutable state)
   - Multiple services တစ်ထဲသော DB ကို directly access
   - Foreign key relationships across service boundaries

3. Synchronous call chains
   - A calls B calls C calls D (5+ deep chains)
   - Any service fail = entire chain fail
```

### Solution

```
Correct Approach: Domain-Driven Design (DDD) Bounded Contexts

User Domain:
+----------------------------------+
| User Service                     |
| - Registration, Auth, Profile    |
| - Owns: users, sessions tables   |
| - API: REST/gRPC                 |
+----------------------------------+

Order Domain:
+----------------------------------+
| Order Service                    |
| - Order lifecycle                |
| - Owns: orders, order_items      |
| - Subscribes to: user.created    |
+----------------------------------+

// Each service:
// ✓ Independent deploy
// ✓ Owns its own database
// ✓ Communicates via well-defined APIs/events
// ✓ Can scale independently
```

---

## ၅၁.၂ Chatty Services & Over-Communication

### ဘာဆိုသည်

Services များသည် single operation ကို complete ပြုရန် မလိုအပ်ဘဲ ထပ်ခါတလဲလဲ communicate ပြုသည်။

### Symptoms

```
Example: Display user profile page
❌ BAD: 
  API Gateway → User Service (get basic info)
  API Gateway → Profile Service (get preferences)
  API Gateway → Orders Service (get order count)
  API Gateway → Friends Service (get friend count)
  API Gateway → Activity Service (get recent activity)
  = 5 sequential calls → High latency, many failure points

✗ N+1 query problem across services
  "Get all users with their order counts"
  → 1 call to get 100 users
  → 100 separate calls to order service (one per user)
  = 101 total calls!

✗ Synchronous service chains > 3 deep
  A → B → C → D → E
  Each hop adds latency + failure probability
```

### Solutions

**1. API Composition / BFF (Backend for Frontend):**
```python
class ProfileBFF:
    """
    Backend for Frontend - aggregates multiple service calls
    """
    
    async def get_user_dashboard(self, user_id):
        # Parallel calls (not sequential!)
        user_data, orders, friends, activity = await asyncio.gather(
            self.user_service.get(user_id),
            self.order_service.get_summary(user_id),
            self.friends_service.get_count(user_id),
            self.activity_service.get_recent(user_id, limit=5)
        )
        
        return {
            "user": user_data,
            "order_count": orders.total,
            "friend_count": friends.count,
            "recent_activity": activity
        }
```

**2. GraphQL for flexible data fetching:**
```graphql
# Client ကို needed data ကိုသာ request ပြုနိုင်ခွင့်ပေး
query UserDashboard($userId: ID!) {
  user(id: $userId) {
    name
    email
    orderCount    # Resolves from Order Service
    friendCount   # Resolves from Friends Service
    recentActivity(limit: 5) {  # Resolves from Activity Service
      type
      timestamp
    }
  }
}
```

**3. Denormalization for common queries:**
```python
# Order တွင် user name ပါ store (denormalize)
# User service call ကို order display အတွင်း မလုပ်ရ
order = {
    "order_id": "ord_123",
    "user_id": "usr_456",
    "user_name": "Ko Aung",  ← Denormalized (stale acceptable)
    "items": [...],
    "total": 50.00
}
```

---

## ၅၁.၃ Shared Databases Across Services

### ဘာဆိုသည်

Multiple services တို့သည် တစ်ထဲသော database ကို directly access ပြုသည်ဆိုသည်မှာ microservices ၏ fundamental principle ကို ချိုးဖောက်ခြင်းဖြစ်သည်။

### Symptoms

```
✗ Service A သည် Service B ၏ table ကို direct SQL query လုပ်သည်
✗ Foreign keys across service boundaries
✗ Database schema change → multiple services break
✗ One service ၏ heavy query = other services slow

// BAD Example:
// Order Service directly queries User table
SELECT u.name, u.email, o.total
FROM users u          ← User Service ၏ table
JOIN orders o         ← Order Service ၏ table
WHERE u.id = ?
```

### Root Cause

Teams တို့သည် "just do a join, it's faster" ဟူသော shortcut ကို ယူကာ service boundaries ကို ဖောက်ဖျားသည်။ Database-level coupling ဖြစ်သောကြောင့် services ကို independently evolve မပြုနိုင်တော့ပါ။

### Solution: Database per Service

```
User Service    →  PostgreSQL (users_db)
Order Service   →  PostgreSQL (orders_db)
Product Service →  MongoDB (products_db)
Analytics       →  ClickHouse (analytics_db)

Data Synchronization via Events:
User Service:  publishes "user.created" event
Order Service: subscribes → stores user_id + user_name (denormalized)
               // Never cross-service DB join again
```

```python
class OrderService:
    
    async def handle_user_created(self, event):
        """
        User service ၏ event ကို subscribe ကာ local copy store
        Avoid cross-service DB join
        """
        await self.db.upsert("local_users", {
            "user_id": event["user_id"],
            "user_name": event["name"],
            "email": event["email"],
            "updated_at": event["timestamp"]
        })
```

---

## ၅၁.၄ Synchronous Chain (Death Star / Spaghetti Dependencies)

### ဘာဆိုသည်

Services များသည် synchronous API calls chains ဖြင့် ချိတ်ဆက်ထားသောကြောင့် failure ကို cascade ဖြစ်ပြီး complex dependency graph ဖြစ်သွားသည်။

### Symptoms (Death Star Diagram)

```
           +---+        +---+
           | A | -----> | B |
           +---+  \     +---+
            |      \     |
            v       v    v
           +---+   +---++---+
           | C | ->| D || E |
           +---+   +---++---+
            |       |   |
            v       v   v
           +---+   +---+
           | F | ->| G |
           +---+   +---+

// Every service depends on every other service
// A → B → D → G (4-deep synchronous chain)
// G is down? Everything is down.
```

### Cascading Failure Example

```python
# BAD: Synchronous chain
class CheckoutService:
    async def checkout(self, order):
        # Sequential sync calls - each adds latency + failure point
        user = await self.user_service.get(order.user_id)        # 50ms
        products = await self.product_service.get(order.items)   # 80ms
        inventory = await self.inventory_service.reserve(order)  # 100ms
        price = await self.pricing_service.calculate(order)      # 60ms
        payment = await self.payment_service.charge(order)       # 200ms
        # Total: 490ms, 5 failure points
        
        # If inventory_service is down:
        # → checkout fails completely
        # → user sees error
        # → order lost
```

### Solution: Async + Circuit Breaker + Fallback

```python
# GOOD: Async + resilience patterns
class CheckoutService:
    
    async def checkout(self, order):
        # Parallel calls where possible
        user_task = asyncio.create_task(self.user_service.get(order.user_id))
        product_task = asyncio.create_task(self.product_service.get(order.items))
        
        user, products = await asyncio.gather(user_task, product_task)
        
        # Circuit breaker wrapping
        try:
            async with self.circuit_breaker("inventory"):
                inventory = await self.inventory_service.reserve(order)
        except CircuitOpenError:
            # Fallback: Queue for later processing
            await self.inventory_queue.enqueue(order)
            inventory = {"status": "pending"}
        
        # Payment is critical - no fallback, but with timeout
        try:
            payment = await asyncio.wait_for(
                self.payment_service.charge(order),
                timeout=5.0
            )
        except asyncio.TimeoutError:
            raise CheckoutError("Payment timeout - please retry")
```

---

## ၅၁.၅ Ignoring the 8 Fallacies of Distributed Computing

Peter Deutsch ၏ "8 Fallacies of Distributed Computing" ကို engineer တိုင်း မသင်မနေ နားလည်ထားရမည်ဖြစ်သည်။

### The 8 Fallacies

```
1. The network is reliable
   Reality: Packets drop, connections timeout, cables fail
   Solution: Timeouts, retries, circuit breakers

2. Latency is zero
   Reality: Cross-datacenter = 50ms, Cross-continent = 100ms+
   Solution: Async where possible, co-locate critical services

3. Bandwidth is infinite
   Reality: 10GbE is shared, large payloads expensive
   Solution: Compression, pagination, event streaming over polling

4. The network is secure
   Reality: Man-in-the-middle, packet sniffing, injection attacks
   Solution: mTLS between services, encrypted payloads

5. Topology doesn't change
   Reality: Kubernetes pod restarts, IP changes, scale in/out
   Solution: Service discovery, health checks, not hardcoded IPs

6. There is one administrator
   Reality: Multiple teams, multiple cloud regions, third-party APIs
   Solution: Automation, Infrastructure as Code, documentation

7. Transport cost is zero
   Reality: Serialization/deserialization CPU cost, data transfer fees
   Solution: Efficient serialization (Protobuf > JSON), caching

8. The network is homogeneous
   Reality: Different OS, languages, protocols, versions
   Solution: API versioning, protocol standardization
```

```python
# Addressing Fallacy #1 & #2: Timeouts + Retries
class ServiceClient:
    
    async def call_with_resilience(self, service_name, endpoint, payload):
        timeout = self.get_timeout(service_name)      # Per-service timeout
        max_retries = 3
        
        for attempt in range(max_retries):
            try:
                response = await asyncio.wait_for(
                    self.http_client.post(endpoint, json=payload),
                    timeout=timeout
                )
                return response
                
            except asyncio.TimeoutError:
                if attempt == max_retries - 1:
                    raise ServiceTimeoutError(f"{service_name} timeout after {max_retries} attempts")
                await asyncio.sleep(2 ** attempt)  # Exponential backoff
                
            except aiohttp.ClientConnectionError:
                # Network unreliable - retry
                await asyncio.sleep(1)
```

---

## ၅၁.၆ Missing Idempotency

### ဘာဆိုသည်

Operations တို့သည် multiple times execute ပြုသောအခါ မတူညီသော results ထွက်သည်ဟူသော situation ဖြစ်သည်။ Distributed systems တွင် retries, duplicate messages, network failures ကြောင့် same operation ကို multiple times receive ပြုနိုင်သည်ဆိုသည်ကို always assume ပြုရမည်ဖြစ်သည်။

### Symptoms

```
✗ User ကို twice charge ပြုမိသည် (payment duplicate)
✗ Order ကို twice create ပြုမိသည် (inventory shortage)
✗ Email ကို twice send ပြုမိသည် (user complaint)
✗ "POST /api/payment" ကို client retry လုပ်သောအခါ ထပ်ကောက်မိသည်
```

### Solution: Idempotency Keys

```python
class PaymentService:
    
    async def charge(self, user_id, amount, idempotency_key):
        """
        Idempotency key ဖြင့် duplicate charges ကာကွယ်
        Client မှ retry ပြုလုပ်သော်လည်း once only charge ဖြစ်ရမည်
        """
        # Check if already processed
        existing = await self.db.get_by_idempotency_key(idempotency_key)
        
        if existing:
            # Same response return (replay-safe)
            return existing.result
        
        # Process payment
        result = await self.payment_gateway.charge(user_id, amount)
        
        # Store with idempotency key (TTL: 24 hours)
        await self.db.store_idempotent_result(
            key=idempotency_key,
            result=result,
            ttl=86400
        )
        
        return result

# Client side: Generate stable idempotency key
import hashlib

def generate_idempotency_key(user_id, order_id, timestamp):
    """
    Deterministic key - retry ၌ same key ထွက်ရမည်
    """
    data = f"{user_id}:{order_id}:{timestamp}"
    return hashlib.sha256(data.encode()).hexdigest()
```

### Idempotent HTTP Methods

```
GET, HEAD, OPTIONS, PUT, DELETE: Idempotent by definition
POST: NOT idempotent by default → Need idempotency key

// API Header convention
POST /api/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

// If same key sent twice:
// First call: Process + return 201 Created
// Second call: Return same response + 200 OK (or 201 with "already_processed" flag)
```

---

## ၅၁.၇ Premature Decomposition

### ဘာဆိုသည်

System ကို prematurely (too early) microservices ခွဲသည်ဆိုသည်မှာ domain ကို မနားလည်မီ၊ traffic pattern မသိမီ service boundaries ကို incorrectly define ပြုကာ ပြင်ရခက်ခဲသော bad architecture ဖြစ်သွားသည်ဟူသောအရာဖြစ်သည်။

### Symptoms

```
✗ 5 engineers ကလေး team ဖြင့် 15+ microservices manage ပြုသည်
✗ Service တစ်ခုတွင် code lines 50 မျှသာ ရှိသည်
✗ Business logic တစ်ခုကို trace လုပ်ရာတွင် 8 services ဖြတ်ရသည်
✗ Local development environment ကို setup လုပ်ရာတွင် 2 hours ကြာသည်
✗ New feature တစ်ခု = 5 services update ပြုလုပ်ရသည်
```

### "Monolith-First" Principle (Martin Fowler)

```
Timeline of Correct Approach:

Phase 1: Monolith (Month 0-12)
  - Domain ကို learn သည်
  - Business logic ကို establish ပြုသည်
  - Traffic patterns ကို understand ပြုသည်
  - Team ကို grow ပြုသည်

Phase 2: Modular Monolith (Month 12-24)
  - Monolith ကို modules ဖြင့် organize ပြုသည်
  - Clear boundaries ကို enforce ပြုသည် (no cross-module DB calls)
  - Still single deployable unit

Phase 3: Extract to Services (Month 24+)
  - High-traffic modules ကို extract ပြုသည်
  - Clearly different teams ၏ modules ကို extract ပြုသည်
  - Evidence-based decomposition

Rule of Thumb:
"If you don't have scale problems, don't have team coordination 
problems, microservices only add complexity without benefit."
```

### Sam Newman ၏ Rule

```
Extract service when:
✓ A component needs to scale independently
✓ A component needs different deployment frequency  
✓ A component needs different technology
✓ A team boundary exists (Conway's Law)
✓ You've proven the domain model is stable

Do NOT extract because:
✗ "Microservices is modern"
✗ "Netflix does it"
✗ "Architecture diagram looks cool"
✗ "We want to use Kubernetes"
```

---

## Anti-Pattern Summary & Detection Checklist

```
+------------------------------------------------------------------+
| Anti-Pattern Detection Checklist                                 |
+------------------------------------------------------------------+
|                                                                  |
| □ Can each service be deployed independently? (Distributed Mon.) |
| □ Are service call chains ≤ 3 deep? (Chatty Services)           |
| □ Does each service own its own database? (Shared DB)           |
| □ Are most inter-service calls async via events? (Death Star)   |
| □ Does code handle network timeouts everywhere? (8 Fallacies)   |
| □ Are all write operations idempotent? (Missing Idempotency)    |
| □ Is decomposition based on evidence? (Premature Decomp.)       |
|                                                                  |
+------------------------------------------------------------------+
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **Distributed Monolith Prevention:** Domain-Driven Design Bounded Contexts ကို ကျင့်သုံးပြီး service ကို data model layer ဖြင့် မဟုတ်ဘဲ business domain ဖြင့် define ပြုရမည်ဖြစ်သည်။ Independent deployment မဖြစ်နိုင်ပါက microservices benefits မရရှိနိုင်ပါ။

2. **Chatty Services Cure:** BFF pattern, parallel async calls, data denormalization ဤ ၃ ခုသည် chatty services ၏ primary solutions ဖြစ်သည်။ Sequential synchronous calls ကို reflex ဖြင့် ရှောင်ကွင်းတတ်ရမည်ဖြစ်သည်။

3. **Database per Service = Non-Negotiable:** Database sharing ကို allow ပြုသောကြောင့် "just one join" သည် architecture debt ၏ ကြီးမားသော source ဖြစ်သည်။ Cross-service data needs ကို events + local denormalized copies ဖြင့် ဖြေရှင်းရမည်ဖြစ်သည်။

4. **8 Fallacies as Design Checklist:** New service ကို design လုပ်တိုင်း "network failure ဖြစ်ပါက ဘာဖြစ်မည်?" "timeout ဖြစ်ပါက?" "duplicate message ရောက်ပါက?" ဟူသော questions ကို ကြောက်ဒဖြေဆိုနိုင်ရမည်ဖြစ်သည်။

5. **Idempotency as Default:** Write operations (payment, order create, email send) များအားလုံးကို idempotent ဖြစ်အောင် design ပြုရမည်ဖြစ်သည်။ "At-least-once delivery" + "Idempotent processing" = "Effectively once" semantics ဖြစ်သည်။

6. **Start with Monolith:** Premature decomposition သည် unnecessary complexity ဖြစ်ပေါ်စေသည်။ Domain ကို ကောင်းစွာ နားလည်ပြီး team coordination problems ရှိသည့်အခါမှသာ services ကို extract ပြုသင့်သည်ဟူသော discipline ကို maintain ပြုရမည်ဖြစ်သည်။

7. **Conway's Law:** Organization structure သည် architecture ကို reflect ပြုသည်ဆိုသော Conway's Law ကို recognize ပြုပြီး service boundaries ကို team boundaries နှင့် align ပြုပါက maintenance friction ကို significantly ကျဆင်းစေနိုင်သည်ဖြစ်သည်။
