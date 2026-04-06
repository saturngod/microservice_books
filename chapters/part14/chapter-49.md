# အခန်း ၄၉: Rate Limiter as a Service

## နိဒါန်း

Rate limiting သည် distributed systems ၏ မရှိမဖြစ် protection mechanism တစ်ခုဖြစ်သည်။ API abuse, DDoS attacks, unintentional traffic spikes တို့မှ downstream services ကို ကာကွယ်ရာတွင် rate limiting သည် first line of defense ဖြစ်သည်။

Microservices architecture တွင် rate limiting ကို multiple levels တွင် apply နိုင်သည်: API Gateway level (global), individual service level (per-service), user level (per-user quotas)။ ဤအခန်းတွင် centralized rate limiter service ကို algorithms အမျိုးမျိုး၊ distributed implementation, Redis-based solution တို့ဖြင့် design ပြုမည်။

---

## ၄၉.၁ Requirements: Global Rate Limiting

### Use Cases

- **API Quota:** Free tier 100 req/hour, Premium 10,000 req/hour, Enterprise 1M req/hour
- **DDoS Protection:** IP-based limiting, endpoint-specific limits
- **Backend Protection:** Payment service 5 attempts/min, Email 100/hour
- **Resource Abuse:** File upload 10/hour, Report generation 1/5min

### Non-Functional Requirements

| Metric | Target |
|--------|--------|
| Rate limit check latency | < 5ms (P99) |
| Accuracy | +/- 0.5% |
| Availability | 99.99% |
| Consistency across DCs | Eventual (acceptable) |
| Storage per 1M users | < 1 GB |

---

## ၄၉.၂ Algorithms: Token Bucket, Leaky Bucket, Fixed Window, Sliding Window Log, Sliding Window Counter

### 1. Fixed Window Counter

```
Timeline: |--window--|--window--|--window--|
          |  0-60s   | 60-120s  |120-180s  |
          |[counter] |[counter] |[counter] |

PROBLEM: Window edge bursting
  Second 59: 100 requests (window 1 max)
  Second 61: 100 requests (window 2 starts)
  -> 200 requests in 2 seconds! (expected max: 100/min)
```

```python
class FixedWindowRateLimiter:

    async def check(self, key):
        window = int(time.time() / self.window_seconds)
        redis_key = f"rl:fw:{key}:{window}"
        count = await self.redis.incr(redis_key)
        if count == 1:
            await self.redis.expire(redis_key, self.window_seconds * 2)
        return RateLimitResult(
            allowed=count <= self.limit,
            current_count=count,
            reset_at=(window + 1) * self.window_seconds
        )
```

### 2. Sliding Window Log

```
Store every request timestamp in sorted set.
Check: remove expired entries, count remaining.

Accurate but memory-intensive:
  Memory per user = O(requests per window)
  10,000 req/hour user = 10,000 timestamps = ~80KB
  1M users = 80 GB (too expensive!)
```

```python
class SlidingWindowLogRateLimiter:

    async def check(self, key):
        now = time.time()
        window_start = now - self.window_seconds
        redis_key = f"rl:swl:{key}"
        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(redis_key, 0, window_start)
        pipe.zcard(redis_key)
        pipe.zadd(redis_key, {str(now): now})
        pipe.expire(redis_key, int(self.window_seconds))
        results = await pipe.execute()
        return results[1] < self.limit
```

### 3. Sliding Window Counter (Recommended)

```
Hybrid of Fixed Window + Sliding approximation
Estimated count = current_count + previous_count x (1 - elapsed_ratio)

Example (limit 100/min):
  Previous window: 80, Current window: 30, Elapsed: 50%
  Estimate = 30 + 80 x 0.5 = 70  -> Allow

Memory: O(1) -- only 2 counters per key
Accuracy: ~0.003% error (excellent)
```

```python
class SlidingWindowCounterRateLimiter:

    async def check(self, key):
        now = time.time()
        current_window = int(now / self.window_seconds) * self.window_seconds
        previous_window = current_window - self.window_seconds
        elapsed_ratio = (now - current_window) / self.window_seconds

        pipe = self.redis.pipeline()
        pipe.get(f"rl:swc:{key}:{int(current_window)}")
        pipe.get(f"rl:swc:{key}:{int(previous_window)}")
        results = await pipe.execute()

        current_count = int(results[0] or 0)
        previous_count = int(results[1] or 0)
        estimated = current_count + previous_count * (1 - elapsed_ratio)

        if estimated >= self.limit:
            return RateLimitResult(allowed=False)

        pipe = self.redis.pipeline()
        pipe.incr(f"rl:swc:{key}:{int(current_window)}")
        pipe.expire(f"rl:swc:{key}:{int(current_window)}", int(self.window_seconds * 2))
        await pipe.execute()
        return RateLimitResult(allowed=True)
```

### 4. Token Bucket

```
Bucket capacity = burst limit (e.g., 100 tokens)
Refill rate = sustained rate (e.g., 10 tokens/second)
Each request consumes 1 token
Bucket empty -> rejected

Use case: Allow burst up to capacity, then sustained rate
```

```python
class TokenBucketRateLimiter:

    SCRIPT = """
    local key = KEYS[1]
    local capacity = tonumber(ARGV[1])
    local refill_rate = tonumber(ARGV[2])
    local requested = tonumber(ARGV[3])
    local now = tonumber(ARGV[4])

    local last = tonumber(redis.call('HGET', key, 'last') or now)
    local tokens = tonumber(redis.call('HGET', key, 'tokens') or capacity)

    local elapsed = now - last
    tokens = math.min(capacity, tokens + (elapsed * refill_rate))

    local allowed = 0
    if tokens >= requested then
        tokens = tokens - requested
        allowed = 1
    end

    redis.call('HSET', key, 'tokens', tokens)
    redis.call('HSET', key, 'last', now)
    redis.call('EXPIRE', key, 3600)
    return {allowed, math.floor(tokens)}
    """

    async def check(self, key, tokens_requested=1):
        result = await self.redis.eval(
            self.SCRIPT, keys=[f"tb:{key}"],
            args=[self.capacity, self.refill_rate, tokens_requested, time.time()]
        )
        return RateLimitResult(allowed=bool(result[0]), remaining=result[1])
```

### 5. Leaky Bucket

```
Queue incoming requests, process at fixed rate (leak rate)
Queue overflow = rejected

Input burst:  |||||||||||||||| (high burst)
Leaky bucket: |  |  |  |  |  | (steady leak)
Output:       #  #  #  #  #   (smooth output)

Use case: Smooth bursty traffic (audio/video, payment processing)
```

### Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst | Best For |
|-----------|--------|----------|-------|----------|
| Fixed Window | O(1) | Low (edge burst) | No | Simple, low-stakes |
| Sliding Log | O(n) | Perfect | No | Strict APIs |
| Sliding Counter | O(1) | High (~0.003%) | No | **General use (recommended)** |
| Token Bucket | O(1) | Good | Yes | APIs with burst allowance |
| Leaky Bucket | O(n) | Good | Smooth | Steady-rate services |

---

## ၄၉.၃ Centralized vs Distributed

### Centralized

```
All services -> Central Redis -> Rate Limit Decision

Pros: Exact global counts, simple, easy config
Cons: SPOF, network latency per request, Redis bottleneck
```

### Distributed

```
Each instance: local counter + periodic sync to central store

Pros: Low latency (local check), no central SPOF
Cons: Temporarily over-allow (sync delay), complex
```

### Hybrid (Recommended)

```python
class HybridRateLimiter:
    """
    Local + Central hybrid
    Local: < 0.1ms, Sync to Redis every 1 second
    """
    def __init__(self, limit, window_seconds, sync_interval=1.0):
        self.limit = limit
        self.local_count = 0
        self.local_budget = limit * 0.8  # Reserve 20% for other instances

    async def check(self, key):
        if self.local_count < self.local_budget:
            self.local_count += 1
            if time.time() - self.last_sync > self.sync_interval:
                asyncio.create_task(self.sync_to_redis(key))
            return True
        return await self.check_redis(key)  # Fallback to Redis
```

---

## ၄၉.၄ Redis Implementation

### Production-Ready Rate Limiter

```python
class ProductionRateLimiter:
    """
    Features: Sliding window counter, Redis Cluster,
    graceful degradation, standard response headers
    """
    async def check_rate_limit(self, identifier, rule):
        try:
            result = await asyncio.wait_for(
                self._check_internal(identifier, rule), timeout=0.005  # 5ms
            )
            return result
        except asyncio.TimeoutError:
            self.metrics.increment("rate_limiter.timeout")
            return RateLimitResult(allowed=True, degraded=True)  # Fail open
        except RedisError:
            self.metrics.increment("rate_limiter.redis_error")
            return RateLimitResult(allowed=True, degraded=True)  # Fail open

    def build_response_headers(self, result, rule):
        return {
            "X-RateLimit-Limit": str(rule.limit),
            "X-RateLimit-Remaining": str(max(0, rule.limit - result.current_count)),
            "X-RateLimit-Reset": str(result.reset_at),
            "Retry-After": str(int(result.retry_after)) if not result.allowed else None
        }
```

---

## ၄၉.၅ Gateway vs Service Level

```
+------------------------------------------------------------------+
|                    Rate Limiting Layers                          |
+------------------------------------------------------------------+
|                                                                  |
|  Layer 1: Network / CDN Level                                    |
|  - IP-based DDoS protection (Cloudflare / AWS Shield)           |
|  - Very rough limits (10,000 req/sec per IP)                     |
|                                                                  |
|  Layer 2: API Gateway Level                                      |
|  - Per API key / user authentication                             |
|  - Global request rate limits                                    |
|  - Apply to ALL downstream services                              |
|                                                                  |
|  Layer 3: Service Level                                          |
|  - Service-specific limits                                       |
|  - Business logic limits (5 password resets/hour)               |
|  - Cross-service rate limit sharing                              |
+------------------------------------------------------------------+

Rule: Gateway = broad limits (protect infrastructure)
      Service = narrow limits (protect business rules)
```

---

## ၄၉.၆ Distributed Clocks & Race Conditions

### Clock Skew Problem

```
Server A clock: 10:00:00.000
Server B clock: 10:00:00.150  (150ms skew)

Fixed Window key: "rl:user123:1000000"
Server A at :59.999 -> window 999
Server B at :00.050 -> window 1000 (different window!)
Result: User gets 2x allowed requests
```

**Solution:** Redis server time ဖြင့် window calculation

```python
async def get_current_window(self):
    redis_time = await self.redis.time()  # (seconds, microseconds)
    server_timestamp = redis_time[0] + redis_time[1] / 1_000_000
    return int(server_timestamp / self.window_seconds) * self.window_seconds
```

### Race Condition: Lua Scripts for Atomicity

```python
# WRONG: Race condition between GET and INCR
count = await redis.get(key)
if count < limit:
    await redis.incr(key)  # Another request between!

# CORRECT: Atomic Lua script
ATOMIC_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local count = tonumber(redis.call('GET', key) or 0)
if count < limit then
    redis.call('INCR', key)
    if count == 0 then redis.call('EXPIRE', key, ARGV[2]) end
    return {1, count + 1}
else
    return {0, count}
end
"""
```

---

## ၄၉.၇ Architecture Diagram

```
+------------------------------------------------------------------+
|                    Rate Limiter Service                          |
+------------------------------------------------------------------+

        Client Request -> API Gateway
                                |
                    +---------------------+
                    |  Rate Limit Check   |
                    |  (Gateway Plugin)   |
                    +---------------------+
                         /         \
                    Fast Path    Slow Path
                   (In-Memory)  (Redis Call)
                         \         /
                    +---------------------+
                    |   Redis Cluster     |
                    |   (Rate Counters)   |
                    +---------------------+
                              |
                    +---------------------+
                    |  Decision Response  |
                    | Allow / Block       |
                    | + Headers           |
                    +---------------------+
                              |
                    Allow -> Forward to Service
                    Block -> 429 Too Many Requests

+------------------------------------------------------------------+
|               Rate Limit Config Service                          |
|  (Rules management, hot reload, per-tier configs)               |
+------------------------------------------------------------------+
```

### Config Example

```yaml
rules:
  - name: "user_api_free_tier"
    match: { user_tier: "free" }
    limits:
      - window: 3600, limit: 100      # 100/hour
      - window: 86400, limit: 500     # 500/day

  - name: "login_endpoint"
    match: { endpoint: "/auth/login" }
    limits:
      - window: 60, limit: 5, key: "ip"
    response:
      status: 429
      message: "Too many login attempts."
      retry_after_header: true
```

---

## အဓိကသင်ခန်းစာများ

1. **Sliding Window Counter** ကို general-purpose use case အတွက် recommend ပြုသည် -- O(1) memory, ~0.003% error
2. **Fail Open:** Redis down ဖြစ်ပါက requests ကို allow ပြုသင့်သည် (legitimate users block မဖြစ်စေရ)
3. **Atomic Lua Scripts:** GET-then-SET race conditions ကို ကာကွယ်ရန် Redis Lua scripts သုံးရသည်
4. **Redis Server Time:** Clock skew avoid ရန် client timestamp အစား Redis TIME command သုံးရသည်
5. **Layered Defense:** Network + Gateway + Service level 3-layer defense ဖြင့် comprehensive coverage ရရှိသည်
6. **Response Headers:** X-RateLimit-* headers ဖြင့် clients ကို rate limit state inform ပြုသည်
