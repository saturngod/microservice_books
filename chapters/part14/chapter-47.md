# အခန်း ၄၇: Ad Serving System (Google Ads / Meta Ads ပုံစံ)

## နိဒါန်း

Digital advertising သည် Google, Meta, Amazon တို့၏ primary revenue source ဖြစ်ပြီး ကမ္ဘာ့ digital economy ၏ အရေးပါသော component တစ်ခုဖြစ်သည်။ Ad serving system design တွင် engineering challenges များမှာ အထူးသဖြင့် ကြီးမားသည်: auction ကို **10 milliseconds** အတွင်း complete ပြုလုပ်ရသည်၊ targeting ကို real-time evaluate ရသည်၊ budget ကို exact ခမ်းနန်းကျ control ပြုရသည်၊ billions of click/impression events ကို accurately track လုပ်ရသည်။

---

## ၄၇.၁ Requirements: RTB Sub-၁၀ms Auction

### Functional Requirements

**Advertiser ဘက်:**
- Ad campaign create, manage, pause
- Targeting criteria (demographics, interests, keywords, location)
- Budget management (daily, lifetime, per-bid limits)
- Performance analytics (impressions, clicks, conversions, ROAS)

**Publisher ဘက်:**
- Ad slot configuration
- Fill rate & revenue optimization
- Brand safety controls

**System ဘက်:**
- Real-time auction (RTB) per ad impression
- Winning ad ကို milliseconds အတွင်း return
- Click & impression tracking (idempotent)
- Fraud detection
- Attribution & conversion tracking

### Non-Functional Requirements

| Metric | Target |
|--------|--------|
| Auction latency (P99) | < 10ms |
| QPS (auction requests) | 5 million/second |
| Ad serving availability | 99.99% |
| Click tracking latency | < 1ms (async) |
| Budget enforcement accuracy | +/- 1% |

### Capacity Estimation

```
Internet users: 5 Billion
Ad impressions per user per day: 500
Total impressions/day: 2.5 Trillion
RTB percentage: ~20%
RTB QPS: 29M x 0.2 = 5.8M QPS

Click events (1% CTR): 25B clicks/day = 289,000 click QPS
```

---

## ၄၇.၂ Domain Decomposition: Ad Exchange, DSP, Targeting, Budget, Delivery, Analytics

```
+------------------------------------------------------------------+
|                       Ad Platform                                |
+------------------------------------------------------------------+
|  +-------------+  +----------+  +-----------+  +------------+   |
|  | Ad Exchange |  | DSP      |  | Targeting |  |  Budget &  |   |
|  | Service     |  | Adapter  |  | Engine    |  |  Pacing    |   |
|  +-------------+  +----------+  +-----------+  +------------+   |
|                                                                  |
|  +-------------+  +----------+  +-----------+  +------------+   |
|  |  Delivery   |  | Click &  |  | Analytics |  | Attribution|   |
|  |  Service    |  | Impression|  | Service  |  | Service    |   |
|  |             |  | Tracker  |  |           |  |            |   |
|  +-------------+  +----------+  +-----------+  +------------+   |
+------------------------------------------------------------------+
```

**Ad Exchange Service** သည် central auction orchestrator ဖြစ်ပြီး bid request fan-out, winner selection (Second-Price Auction), win notification တို့ကို ဆောင်ရွက်သည်။

**DSP (Demand-Side Platform) Adapter** သည် external DSP communication, OpenRTB protocol serialization, timeout management (deadline: 100ms) တို့ကို handle လုပ်သည်။

**Targeting Engine** သည် user segment matching, contextual targeting (page content), behavioral targeting (user history) တို့ကို real-time evaluate လုပ်သည်။

**Budget & Pacing Service** သည် real-time budget checking, spend throttling (even distribution throughout day), budget depletion detection တို့ကို manage လုပ်သည်။

**Click & Impression Tracker** သည် high-throughput event ingestion, idempotent event processing, real-time aggregation တို့ကို ပြုလုပ်သည်။

**Attribution Service** သည် click-to-conversion attribution, multi-touch attribution models, conversion window management ကို handle လုပ်သည်။

---

## ၄၇.၃ RTB Pipeline

### RTB Timeline (100ms Total Budget)

```
0ms   - Publisher website loads, user visits page
10ms  - Ad tag fires, Bid Request created
20ms  - Targeting Engine: User segment lookup (Redis, < 2ms)
30ms  - Eligible ads filtered (campaign targeting match)
40ms  - Budget check (Redis Lua script, < 1ms)
50ms  - Bid price calculation (base bid x quality score x user value)
70ms  - Auction: Collect all bids, Second-price auction
80ms  - Win notification to winner (async)
100ms - Ad creative served to user browser
```

### RTB Implementation

```python
class AdExchange:

    async def handle_bid_request(self, bid_request: BidRequest):
        start_time = time.monotonic()

        # Step 1: User segment lookup (< 5ms hard timeout)
        user_segments = await self.targeting_engine.get_user_segments(
            user_id=bid_request.user_id, timeout_ms=5
        )

        # Step 2: Find eligible ads (parallel, < 10ms)
        eligible_ads = await self.find_eligible_ads(
            user_segments=user_segments,
            page_url=bid_request.page_url,
            ad_slot=bid_request.ad_slot,
            timeout_ms=10
        )

        if not eligible_ads:
            return BidResponse(winner=None, reason="no_eligible_ads")

        # Step 3: Calculate bids (parallel for all candidates, max 50)
        bid_tasks = [
            self.calculate_bid(ad, bid_request, user_segments)
            for ad in eligible_ads[:50]
        ]
        bids = await asyncio.gather(*bid_tasks, return_exceptions=True)
        valid_bids = [b for b in bids if isinstance(b, Bid) and b.amount > 0]

        if not valid_bids:
            return BidResponse(winner=None, reason="no_valid_bids")

        # Step 4: Second-price auction
        winner, clearing_price = self.run_auction(valid_bids)

        # Step 5: Async tracking (non-blocking)
        asyncio.create_task(self.track_impression(bid_request, winner, clearing_price))
        asyncio.create_task(self.deduct_budget(winner.campaign_id, clearing_price))

        elapsed_ms = (time.monotonic() - start_time) * 1000
        return BidResponse(winner=winner, clearing_price=clearing_price, latency_ms=elapsed_ms)

    def run_auction(self, bids):
        """
        Second-price auction (Vickrey Auction)
        Winner = Highest bidder
        Clearing price = Second highest bid + $0.01
        """
        sorted_bids = sorted(bids, key=lambda b: b.amount, reverse=True)
        winner = sorted_bids[0]
        if len(sorted_bids) > 1:
            clearing_price = sorted_bids[1].amount + 0.01
        else:
            clearing_price = winner.floor_price
        return winner, clearing_price
```

---

## ၄၇.၄ Targeting Engine

### User Segment Lookup (Sub-2ms)

```python
class TargetingEngine:

    async def get_user_segments(self, user_id: str, timeout_ms: int = 5):
        # L1: In-process memory cache (sub-millisecond)
        cached = self.local_cache.get(user_id)
        if cached and not cached.is_expired():
            return cached.segments

        # L2: Redis (~1ms)
        redis_key = f"user:segments:{user_id}"
        segments_json = await self.redis.get(redis_key)
        if segments_json:
            segments = json.loads(segments_json)
            self.local_cache.set(user_id, segments, ttl_seconds=60)
            return segments

        # L3: Database (fallback, cold cache only)
        user_profile = await self.user_db.get(user_id)
        segments = self.compute_segments(user_profile)
        await self.redis.setex(redis_key, 3600, json.dumps(segments))
        return segments

    def check_targeting_match(self, ad_targeting, user_segments, context):
        # Age, Geographic, Interest/Segment, Keyword targeting checks
        if ad_targeting.age_range:
            if not (ad_targeting.age_range.min <= context.user_age <= ad_targeting.age_range.max):
                return False
        if ad_targeting.geo_targets:
            if context.user_country not in ad_targeting.geo_targets:
                return False
        if ad_targeting.interest_segments:
            if not set(ad_targeting.interest_segments) & set(user_segments):
                return False
        return True
```

---

## ၄၇.၅ Budget & Pacing Control

Advertiser ၏ daily budget ($1,000) ကို 24 hours တစ်လျှောက် evenly distribute မလုပ်ပါက budget သည် morning peak တွင် ကုန်သွားနိုင်သည်။ Token Bucket-based pacing ဖြင့် ဖြေရှင်းသည်။

```python
class BudgetPacingService:

    async def check_budget_available(self, campaign_id, bid_amount):
        """
        Redis Lua script ဖြင့် atomic budget check + deduction (< 1ms)
        """
        script = """
        local key = KEYS[1]
        local bid = tonumber(ARGV[1])
        local now = tonumber(ARGV[2])
        local refill_rate = tonumber(ARGV[3])
        local max_budget = tonumber(ARGV[4])

        local last_refill = tonumber(redis.call('HGET', key, 'last_refill') or now)
        local current_budget = tonumber(redis.call('HGET', key, 'current') or max_budget)

        local elapsed = now - last_refill
        local refill_amount = elapsed * refill_rate
        current_budget = math.min(max_budget, current_budget + refill_amount)

        if current_budget < bid then
            redis.call('HSET', key, 'last_refill', now)
            redis.call('HSET', key, 'current', current_budget)
            return 0
        end

        current_budget = current_budget - bid
        redis.call('HSET', key, 'last_refill', now)
        redis.call('HSET', key, 'current', current_budget)
        return 1
        """
        campaign = await self.campaign_cache.get(campaign_id)
        refill_rate = campaign.daily_budget / 86400
        result = await self.redis.eval(
            script,
            keys=[f"budget:pacing:{campaign_id}"],
            args=[bid_amount, time.time(), refill_rate, campaign.daily_budget]
        )
        return bool(result)
```

---

## ၄၇.၆ Click & Impression Tracking

```python
class EventTracker:

    async def track_impression(self, event: ImpressionEvent):
        """
        Idempotent impression tracking using Bloom filter + Kafka
        """
        idempotency_key = hashlib.sha256(
            f"{event.ad_id}:{event.user_id}:{event.request_id}".encode()
        ).hexdigest()

        # Bloom filter: fast duplicate check (< 0.1ms)
        if self.bloom_filter.contains(idempotency_key):
            return  # Probable duplicate, skip
        self.bloom_filter.add(idempotency_key)

        # Kafka publish (async, non-blocking)
        await self.kafka.publish(
            topic="ad.impressions",
            key=event.campaign_id,
            value={
                "event_type": "IMPRESSION",
                "ad_id": event.ad_id,
                "campaign_id": event.campaign_id,
                "clearing_price": event.clearing_price,
                "timestamp": event.timestamp,
                "idempotency_key": idempotency_key
            }
        )

    async def track_click(self, click_token):
        """
        Click tracking with encrypted token validation
        """
        click_data = self.decode_click_token(click_token)
        if not self.is_valid_click(click_data):
            return None
        await self.kafka.publish("ad.clicks", key=click_data["campaign_id"],
                                  value={**click_data, "click_timestamp": time.time()})
        return click_data["destination_url"]
```

---

## ၄၇.၇ Frequency Capping (Redis + HyperLogLog)

Frequency capping သည် user တစ်ဦးကို တူညီသော ad ကို သတ်မှတ်ထားသည့် threshold ထက် မပိုဘဲ ပြသည်။

```python
class FrequencyCapper:

    async def check_frequency_cap(self, user_id, ad_id, cap_type):
        if cap_type == "hourly":
            window_key = datetime.now().strftime("%Y%m%d%H")
            cap_limit = self.get_cap(ad_id, "hourly")  # e.g., 3/hour
        elif cap_type == "daily":
            window_key = datetime.now().strftime("%Y%m%d")
            cap_limit = self.get_cap(ad_id, "daily")   # e.g., 10/day

        redis_key = f"freq_cap:{ad_id}:{user_id}:{window_key}"
        current_count = await self.redis.incr(redis_key)
        if current_count == 1:
            await self.redis.expire(redis_key, self.get_ttl(cap_type))
        return current_count <= cap_limit

    async def estimate_unique_users_reached(self, ad_id, date):
        """
        HyperLogLog: unique users estimate (memory efficient)
        Exact count: 10M users x 8 bytes = 80 MB per ad
        HyperLogLog: ~12 KB with 0.81% standard error
        """
        hll_key = f"hll:ad:{ad_id}:{date}"
        await self.redis.pfadd(hll_key, user_id)
        return await self.redis.pfcount(hll_key)
```

---

## ၄၇.၈ Attribution & Conversion Tracking

```
User Journey Example:
    Day 1: Sees Ad A (impression) -> click -> visits website
    Day 2: Sees Ad B (impression) -> no click
    Day 3: Sees Ad A again -> click
    Day 5: Makes a purchase ($50)

Attribution Models:
- Last Click: Ad A (Day 3) gets 100% credit
- First Click: Ad A (Day 1) gets 100% credit
- Linear: Equal distribution across all touchpoints
- Time Decay: Recent interactions get more credit
- Data-Driven (ML): Based on historical conversion patterns
```

```python
class AttributionService:

    async def attribute_conversion(self, conversion_event, window_days=30):
        cutoff = conversion_event.timestamp - timedelta(days=window_days)

        touchpoints = await self.touchpoint_db.get(
            user_id=conversion_event.user_id,
            after=cutoff, before=conversion_event.timestamp
        )

        if not touchpoints:
            return []

        model = self.get_attribution_model(conversion_event.advertiser_id)
        attributed = model.distribute_credit(touchpoints, conversion_event.value)

        for attr in attributed:
            await self.kafka.publish("ad.attributions", {
                "campaign_id": attr.campaign_id,
                "conversion_value": attr.credit_value,
                "conversion_type": conversion_event.type,
            })
        return attributed
```

---

## ၄၇.၉ Architecture Diagram

```
+------------------------------------------------------------------+
|                         Ad Serving Platform                      |
+------------------------------------------------------------------+
Publisher Page Load -> Bid Request
                                |
                    +-----------+-----------+
                    |       Ad Exchange     |
                    |   (Auction Engine)    |
                    +-----------+-----------+
                        /       |       \
                 Targeting  Budget    Fraud
                 Engine     Check     Filter
                   |           |         |
                Redis        Redis    ML Model
                (segments)  (budget)
                    \       |       /
                    +-----------+-----------+
                    |   Winner Determined   |
                    |   (Second Price)      |
                    +-----------+-----------+
                                |
                    +-----------+-----------+
                    |                       |
                    v                       v
            Win Notification         Ad Creative Served
            (async, Kafka)           to User Browser
                    |
            +-------+-------+
            |               |
            v               v
    Impression           Budget
    Tracker             Deduction
    (Kafka)             (Redis)
            |
            v
    Aggregation Pipeline (Flink / Spark)
            |
            v
    Analytics DB (ClickHouse)
            |
    Attribution Service
```

---

## အဓိကသင်ခန်းစာများ

1. **Sub-10ms Auction:** Database queries ကို auction critical path မှ ဖယ်ရှားပြီး in-process cache + Redis ဖြင့် sub-2ms lookups ပြုလုပ်ရသည်
2. **Second-Price Auction:** Truthful bidding ကို incentivize ပြုပြီး overbidding ကို discourage လုပ်သည်
3. **Token Bucket Pacing:** Daily budget ကို Redis Lua script atomic operation ဖြင့် evenly distribute ပြုသည်
4. **Bloom Filter + Kafka:** Idempotent tracking ဖြင့် duplicate impressions ကို eliminate ပြုသည်
5. **HyperLogLog:** 12KB memory ဖြင့် unique user reach ကို 0.81% error ဖြင့် estimate ပြုသည်
6. **Attribution Models:** Last-click ထက် data-driven attribution သည် upper-funnel ads ၏ value ကို ပိုမိုတိကျစွာ measure ပြုသည်
