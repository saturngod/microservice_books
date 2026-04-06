# အခန်း ၂၁: Resilience Patterns (ခံနိုင်ရည်ရှိမှု ပုံစံများ)

## နိဒါန်း

Distributed systems တွင် failures များသည် မရှောင်လွှဲနိုင်ပါ - hardware failure, network partition, service overload, dependency timeout - ဤအရာများသည် "if" မဟုတ်ဘဲ "when" ဆိုသည့် မေးခွန်းသာ ဖြစ်ပါသည်။ Resilient system ဆိုသည်မှာ failure တစ်ခုကြောင့် system တစ်ခုလုံး ပြိုကျမသွားဘဲ gracefully degrade ဖြစ်နိုင်သော system ဖြစ်သည်။ Netflix ကဲ့သို့သော company များသည် ဤ patterns များကို production scale တွင် ကျင့်သုံးပြီး battle-tested ဖြစ်ပြီးဖြစ်ပါသည်။ ဤအခန်းတွင် circuit breaker, retry, bulkhead, rate limiting စသည့် resilience patterns များကို code ဥပမာများဖြင့် လေ့လာပါမည်။

---

## ၂၁.၁ Circuit Breaker (Hystrix, Resilience၄j)

**Circuit Breaker** pattern သည် electrical circuit breaker ကဲ့သို့ အလုပ်လုပ်ပါသည်။ Downstream service fail ဖြစ်နေလျှင် request များကို ဆက်မပို့ဘဲ circuit ကို "open" လုပ်ပြီး fail fast ပြန်ပေးသည်။ ဤသို့ဖြင့် cascading failure ကို ကာကွယ်သည်။

```
Circuit Breaker State Machine:

  +--------+    failure threshold    +--------+    timeout    +-----------+
  | CLOSED | ----reached---------> | OPEN   | -----------> | HALF-OPEN |
  +--------+                        +--------+              +-----------+
      ^                                  ^                       |
      |            success               |     failure           |
      +---- <------ (reset) <-----------+---- <-----------------+
                                         |     success
                                         +---- (stay open) <----+
```

```python
# Resilience4j-style Circuit Breaker implementation
import time
from enum import Enum
from collections import deque

class CircuitState(Enum):
    CLOSED = "CLOSED"          # ပုံမှန် အလုပ်လုပ်နေခြင်း
    OPEN = "OPEN"              # Request များ block ထားခြင်း
    HALF_OPEN = "HALF_OPEN"    # စမ်းသပ် request ပို့နေခြင်း

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30, 
                 success_threshold=3):
        self.failure_threshold = failure_threshold    # Fail ခံနိုင်သော အကြိမ်ရေ
        self.recovery_timeout = recovery_timeout      # Open state မှ half-open သို့ ပြောင်းရန် စောင့်ချိန်
        self.success_threshold = success_threshold    # Half-open မှ closed သို့ ပြောင်းရန် လိုအပ်သော success
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None

    def call(self, func, *args, **kwargs):
        """Circuit breaker ဖြင့် function ကို ခေါ်ယူခြင်း"""
        if self.state == CircuitState.OPEN:
            # Recovery timeout ပြီးလျှင် half-open သို့ ပြောင်းခြင်း
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
            else:
                raise CircuitOpenException("Circuit is OPEN - fail fast")

        try:
            result = func(*args, **kwargs)  # Downstream service ကို call လုပ်ခြင်း
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        """Call အောင်မြင်သည့်အခါ state update လုပ်ခြင်း"""
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = CircuitState.CLOSED    # ပုံမှန် ပြန်ဖြစ်ခြင်း
                self.failure_count = 0
        self.failure_count = 0

    def _on_failure(self):
        """Call fail ဖြစ်သည့်အခါ state update လုပ်ခြင်း"""
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN           # Circuit ဖွင့်ခြင်း

class CircuitOpenException(Exception):
    pass

# အသုံးပြုပုံ
cb = CircuitBreaker(failure_threshold=3, recovery_timeout=10)
try:
    result = cb.call(lambda: requests.get("http://payment-service/api/pay"))
except CircuitOpenException:
    print("Payment service unavailable - using fallback")  # Fallback logic
```

---

## ၂၁.၂ Retry with Exponential Backoff & Jitter

Transient failure (ယာယီ ချို့ယွင်းမှု) ဖြစ်သည့်အခါ request ကို retry လုပ်ခြင်းသည် ရိုးရှင်းသော resilience pattern ဖြစ်ပါသည်။ သို့သော် retry storm (retry ကို တစ်ပြိုင်နက် လုပ်ခြင်း) ကို ရှောင်ကြဉ်ရန် **exponential backoff** နှင့် **jitter** ကို ပေါင်းစပ် အသုံးပြုရပါသည်။

```
Exponential Backoff with Jitter:

Attempt 1: wait 0s
Attempt 2: wait ~1s   (base * 2^1 + random jitter)
Attempt 3: wait ~2s   (base * 2^2 + random jitter)
Attempt 4: wait ~4s   (base * 2^3 + random jitter)
Attempt 5: wait ~8s   (base * 2^4 + random jitter)
           Give up     (max retries reached)

+---+  +---+     +---+        +---+              +---+
| 1 |  | 2 |     | 3 |        | 4 |              | 5 |
+---+  +---+     +---+        +---+              +---+
|-----|---1s---|----2s------|----- 4s ----------|---8s---|
```

```python
import time
import random

def retry_with_backoff(func, max_retries=5, base_delay=1.0, max_delay=60.0):
    """
    Exponential backoff နှင့် jitter ပါဝင်သော retry function
    
    Parameters:
        func: retry လုပ်မည့် function
        max_retries: အများဆုံး retry အကြိမ်ရေ
        base_delay: ပထမ delay (စက္ကန့်)
        max_delay: အများဆုံး delay (စက္ကန့်)
    """
    for attempt in range(max_retries):
        try:
            return func()  # Function ကို call လုပ်ခြင်း
        except (ConnectionError, TimeoutError) as e:
            if attempt == max_retries - 1:
                raise  # နောက်ဆုံး attempt ဖြစ်လျှင် exception ပြန်ပေးခြင်း

            # Exponential backoff ဖြင့် delay တွက်ခြင်း
            delay = min(base_delay * (2 ** attempt), max_delay)
            # Full jitter ထည့်ခြင်း - thundering herd problem ကို ရှောင်ခြင်း
            jitter = random.uniform(0, delay)
            actual_delay = jitter

            print(f"Attempt {attempt + 1} failed. Retrying in {actual_delay:.2f}s...")
            time.sleep(actual_delay)

# Decorator pattern ဖြင့် retry
from functools import wraps

def retryable(max_retries=3, backoff_base=1.0):
    """Retry decorator - function ပေါ်တွင် တပ်ဆင် အသုံးပြုခြင်း"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            return retry_with_backoff(
                lambda: func(*args, **kwargs),
                max_retries=max_retries,
                base_delay=backoff_base
            )
        return wrapper
    return decorator

@retryable(max_retries=3, backoff_base=0.5)
def call_payment_service(order_id):
    """Payment service ကို ခေါ်ခြင်း - failure ဖြစ်လျှင် auto retry"""
    response = requests.post(f"http://payment-service/api/pay/{order_id}")
    response.raise_for_status()
    return response.json()
```

---

## ၂၁.၃ Bulkhead Pattern

**Bulkhead** pattern သည် သင်္ဘော၏ bulkhead (အခန်းခြား) ကဲ့သို့ resources များကို isolate လုပ်ခြင်း ဖြစ်ပါသည်။ Dependency တစ်ခု fail ဖြစ်သည့်အခါ ထို dependency အတွက် allocate ထားသော resources ကိုသာ ထိခိုက်ပြီး system ၏ ကျန် အစိတ်အပိုင်းများ ဆက်လက် အလုပ်လုပ်နိုင်သည်။

```
Without Bulkhead:                    With Bulkhead:
+------------------------+          +------------------------+
| Shared Thread Pool     |          | Thread Pool A (10)     |
| (50 threads total)     |          | -> Service A calls     |
|                        |          +------------------------+
| Service A (slow) uses  |          | Thread Pool B (20)     |
| all 50 threads!        |          | -> Service B calls     |
| Service B: BLOCKED!    |          +------------------------+
| Service C: BLOCKED!    |          | Thread Pool C (20)     |
+------------------------+          | -> Service C calls     |
                                    +------------------------+
```

```python
# Bulkhead pattern - Thread pool isolation
from concurrent.futures import ThreadPoolExecutor, TimeoutError
from functools import wraps

class Bulkhead:
    def __init__(self, name, max_concurrent, max_wait_time=5):
        """
        Bulkhead ဖန်တီးခြင်း
        name: Bulkhead အမည်
        max_concurrent: တစ်ပြိုင်နက် ခွင့်ပြုသော request အရေအတွက်
        max_wait_time: Queue တွင် စောင့်ဆိုင်းရမည့် အများဆုံး အချိန် (စက္ကန့်)
        """
        self.name = name
        self.executor = ThreadPoolExecutor(
            max_workers=max_concurrent,
            thread_name_prefix=f"bulkhead-{name}"
        )
        self.max_wait_time = max_wait_time

    def execute(self, func, *args, **kwargs):
        """Bulkhead အတွင်း function execute လုပ်ခြင်း"""
        future = self.executor.submit(func, *args, **kwargs)
        try:
            return future.result(timeout=self.max_wait_time)
        except TimeoutError:
            future.cancel()
            raise BulkheadFullException(
                f"Bulkhead '{self.name}' is full - request rejected"
            )

class BulkheadFullException(Exception):
    pass

# Service တစ်ခုချင်းစီအတွက် သီးသန့် bulkhead ဖန်တီးခြင်း
payment_bulkhead = Bulkhead("payment", max_concurrent=10)
inventory_bulkhead = Bulkhead("inventory", max_concurrent=20)

# Payment service call - payment bulkhead အတွင်း execute လုပ်ခြင်း
try:
    result = payment_bulkhead.execute(call_payment_service, order_id=123)
except BulkheadFullException:
    print("Payment bulkhead full - returning cached response")
```

---

## ၂၁.၄ Timeout & Deadline Propagation

Distributed system တွင် request timeout ကို မှန်ကန်စွာ manage မလုပ်လျှင် resource leak ဖြစ်နိုင်သည်။ **Deadline propagation** သည် request ၏ overall deadline ကို downstream services အားလုံးသို့ ဖြန့်ဝေခြင်း ဖြစ်ပါသည်။

```
Client (timeout: 5s)
  |
  v
Gateway (remaining: 5s, own processing: 0.5s)
  |  deadline = now + 4.5s
  v
Order Service (remaining: 4.5s, own processing: 1s)
  |  deadline = now + 3.5s
  v
Payment Service (remaining: 3.5s)
  |  deadline = now + 3.5s
  v
Bank API (remaining: 3.5s)
```

```python
import time
import requests

class DeadlineContext:
    """Request deadline ကို propagate လုပ်ရန် context object"""
    
    def __init__(self, timeout_seconds):
        self.deadline = time.time() + timeout_seconds  # Absolute deadline

    def remaining(self):
        """ကျန်ရှိသော အချိန် (စက္ကန့်)"""
        remaining = self.deadline - time.time()
        if remaining <= 0:
            raise DeadlineExceededException("Deadline exceeded")
        return remaining

    def make_request(self, url, **kwargs):
        """Deadline-aware HTTP request"""
        remaining = self.remaining()
        # Downstream service သို့ remaining deadline ကို header ဖြင့် ပို့ခြင်း
        headers = kwargs.pop('headers', {})
        headers['X-Request-Deadline'] = str(self.deadline)
        headers['X-Request-Timeout-Ms'] = str(int(remaining * 1000))
        
        return requests.get(
            url, 
            timeout=remaining,  # Remaining time ကို timeout အဖြစ် သတ်မှတ်ခြင်း
            headers=headers,
            **kwargs
        )

class DeadlineExceededException(Exception):
    pass

# အသုံးပြုပုံ - overall deadline 5 စက္ကန့်
ctx = DeadlineContext(timeout_seconds=5)
try:
    order = ctx.make_request("http://order-service/api/orders/123")
    payment = ctx.make_request("http://payment-service/api/pay")  # ကျန်ချိန်ဖြင့်
except DeadlineExceededException:
    print("Overall request deadline exceeded")
```

---

## ၂၁.၅ Rate Limiting & Throttling

**Rate Limiting** သည် service သို့ ဝင်ရောက်လာသော request အရေအတွက်ကို ကန့်သတ်ခြင်း ဖြစ်ပါသည်။ DDoS attack ကာကွယ်ခြင်း၊ resource protection၊ fair usage enforcement တို့အတွက် အသုံးပြုသည်။

### Token Bucket Algorithm

```
Token Bucket:
+---------------------------+
| Bucket (max: 10 tokens)   |
| [*][*][*][*][*][*][ ][ ]  |  <- 6 tokens ကျန်ရှိခြင်း
+---------------------------+
     ^                |
     |                v
  Refill           Consume
  (2/sec)         (1 per request)
```

```python
import time
import threading

class TokenBucket:
    """Token Bucket rate limiter"""
    
    def __init__(self, capacity, refill_rate):
        """
        capacity: Bucket ၏ အများဆုံး token အရေအတွက်
        refill_rate: တစ်စက္ကန့်လျှင် ပြန်ဖြည့်သော token အရေအတွက်
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity             # အစတွင် bucket ပြည့်နေခြင်း
        self.last_refill = time.time()
        self.lock = threading.Lock()

    def allow_request(self):
        """Request ခွင့်ပြုသည်/မပြုသည် စစ်ဆေးခြင်း"""
        with self.lock:
            self._refill()
            if self.tokens >= 1:
                self.tokens -= 1           # Token တစ်ခု consume လုပ်ခြင်း
                return True
            return False                    # Token မရှိ - request ပယ်ချခြင်း

    def _refill(self):
        """ကုန်ဆုံးသော token များကို ပြန်ဖြည့်ခြင်း"""
        now = time.time()
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now

# Sliding Window Counter (ပိုတိကျသော rate limiting)
class SlidingWindowCounter:
    """Sliding window rate limiter - Redis ဖြင့် distributed implementation"""
    
    def __init__(self, redis_client, limit, window_seconds):
        self.redis = redis_client
        self.limit = limit                 # Window အတွင်း ခွင့်ပြုသော request အရေအတွက်
        self.window = window_seconds       # Window size (စက္ကန့်)

    def allow_request(self, client_id):
        """Client ID အလိုက် rate limit စစ်ဆေးခြင်း"""
        now = time.time()
        window_start = now - self.window
        key = f"rate_limit:{client_id}"
        
        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(key, 0, window_start)  # Old entries ဖယ်ရှားခြင်း
        pipe.zadd(key, {str(now): now})               # Current request ထည့်ခြင်း
        pipe.zcard(key)                                # Window အတွင်း request count
        pipe.expire(key, self.window)                  # Key expiry သတ်မှတ်ခြင်း
        results = pipe.execute()
        
        request_count = results[2]
        return request_count <= self.limit

# Rate limiter middleware
limiter = TokenBucket(capacity=100, refill_rate=10)  # 100 burst, 10/sec sustained

def rate_limit_middleware(request):
    if not limiter.allow_request():
        return {"error": "Rate limit exceeded"}, 429  # HTTP 429 Too Many Requests
    return handle_request(request)
```

---

## ၂၁.၆ Graceful Degradation & Fallback

System ၏ dependency fail ဖြစ်သည့်အခါ completely fail ဖြစ်မည့်အစား reduced functionality ဖြင့် ဆက်လက် service ပေးခြင်းကို **graceful degradation** ဟု ခေါ်ပါသည်။

```
Normal Operation:                    Degraded Operation:
+------------------+                 +------------------+
| Product Page     |                 | Product Page     |
|                  |                 |                  |
| [Product Info]   | <-- DB          | [Product Info]   | <-- DB
| [Reviews  ]      | <-- Review Svc  | [Reviews: N/A]   | <-- FALLBACK (cached)
| [Recommend ]     | <-- ML Service  | [Top Sellers]    | <-- FALLBACK (static)
| [Live Price]     | <-- Price Svc   | [Cached Price]   | <-- FALLBACK (cache)
+------------------+                 +------------------+
```

```python
# Fallback pattern implementation
from functools import wraps

def with_fallback(fallback_func):
    """Fallback decorator - primary function fail ဖြစ်လျှင် fallback function ကို ခေါ်ခြင်း"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            try:
                return func(*args, **kwargs)
            except Exception as e:
                print(f"Primary failed: {e}. Using fallback.")
                return fallback_func(*args, **kwargs)
        return wrapper
    return decorator

# Cache-based fallback
def get_cached_recommendations(user_id):
    """ML service fail ဖြစ်လျှင် cached recommendations ပြန်ပေးခြင်း"""
    cached = redis_client.get(f"recommendations:{user_id}")
    if cached:
        return json.loads(cached)
    return get_top_sellers()  # Cache မရှိလျှင် generic top sellers ပြခြင်း

@with_fallback(get_cached_recommendations)
def get_recommendations(user_id):
    """ML service မှ personalized recommendations ရယူခြင်း"""
    response = requests.get(
        f"http://recommendation-service/api/recommend/{user_id}",
        timeout=2
    )
    response.raise_for_status()
    data = response.json()
    # Success ဖြစ်လျှင် cache ထဲ သိမ်းခြင်း (fallback အတွက်)
    redis_client.setex(f"recommendations:{user_id}", 3600, json.dumps(data))
    return data
```

---

## အဓိက အချက်များ (Key Takeaways)

- **Circuit Breaker** သည် cascading failure ကို ကာကွယ်ပြီး fail fast ပြန်ပေးသည်။ Closed -> Open -> Half-Open state machine ဖြင့် အလုပ်လုပ်သည်
- **Retry with Exponential Backoff + Jitter** သည် transient failure အတွက် သင့်လျော်ပြီး thundering herd ကို ကာကွယ်သည်
- **Bulkhead** သည် resource isolation ဖြင့် failure blast radius ကို ကန့်သတ်သည်
- **Deadline Propagation** ဖြင့် downstream services အားလုံးတွင် overall timeout ကို enforce လုပ်နိုင်သည်
- **Rate Limiting** (Token Bucket, Sliding Window) ဖြင့် service overload ကို ကာကွယ်နိုင်သည်
- **Graceful Degradation** ဖြင့် dependency fail ဖြစ်လျှင်လည်း partial functionality ဆက်ပေးနိုင်သည်
