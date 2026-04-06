# အခန်း ၄၆: Notification System

## နိဒါန်း

Notification system သည် modern applications ၏ မရှိမဖြစ် component တစ်ခုဖြစ်သည်။ Bank OTP မှ marketing emails အထိ၊ ride-sharing driver alerts မှ social media likes အထိ — notifications သည် real-time user engagement ၏ backbone ဖြစ်သည်။ Facebook, Google, Uber ကဲ့သို့ large-scale platforms တွင် တစ်နေ့လျှင် notifications ဘီလီယံကျော် ပေးပို့ရသည်။

ဤ system ကို design လုပ်ရာတွင် multiple channels (Push, SMS, Email, WhatsApp) support, channel-specific rate limits, delivery failures retry, user preferences respect စသည့် challenges များစွာ ကြုံတွေ့ရသည်။

---

## ၄၆.၁ Requirements: Push, SMS, Email, In-App, WhatsApp at ၁B+/day

### Functional Requirements

**Notification Types:**
- **Push Notifications:** Mobile devices (Android FCM, iOS APNS)
- **SMS:** Transactional OTPs, alerts (Twilio, AWS SNS)
- **Email:** Marketing, transactional (SendGrid, SES)
- **In-App:** Real-time notifications within the app (WebSocket)
- **WhatsApp:** Business messaging (WhatsApp Business API)

**Core Features:**
- Template-based notification creation (variable substitution, i18n)
- User preference management (opt-in/opt-out per channel)
- Do-Not-Disturb (DND) hours support
- Delivery status tracking (sent, delivered, read, failed)
- Retry on failure with exponential backoff
- Analytics & reporting (delivery rates, open rates, CTR)

### Non-Functional Requirements

| Metric | Target |
|--------|--------|
| Daily volume | 1 Billion+ notifications |
| Push notification latency | < 1 second (P99) |
| SMS delivery time | < 30 seconds |
| Email delivery | < 5 minutes |
| System availability | 99.9% |
| Delivery success rate | > 99% (transactional) |

### Scale Estimation

```
Daily volume breakdown:
Push notifications:    600M / day = 7,000 QPS (average), 70,000 QPS (peak 10x)
Email notifications:   300M / day = 3,500 QPS
SMS notifications:      50M / day =   580 QPS
In-app notifications:  100M / day = 1,160 QPS

Storage:
Notification records: 1B x 500 bytes = 500 GB/day
90-day retention: ~45 TB
```

---

## ၄၆.၂ Domain Decomposition: Notification, Template, Preference, Channel Adapters, Delivery Tracker

```
+------------------------------------------------------------------+
|                     Notification Platform                        |
+------------------------------------------------------------------+
|                                                                  |
|  +-----------------+        +---------------------------+        |
|  | Notification    |        |    Template Service       |        |
|  | API Service     |        | (Template CRUD, Rendering)|        |
|  +-----------------+        +---------------------------+        |
|                                                                  |
|  +-----------------+        +---------------------------+        |
|  | Preference      |        |    Delivery Tracker       |        |
|  | Service         |        |    Service                |        |
|  +-----------------+        +---------------------------+        |
|                                                                  |
|  Channel Adapters:                                               |
|  +---------+  +-------+  +----------+  +------+  +----------+  |
|  |  FCM    |  | APNS  |  |  Twilio  |  |SendG.|  | WhatsApp |  |
|  | Adapter |  |Adapter|  |  Adapter |  |Adapt.|  |  Adapter |  |
|  +---------+  +-------+  +----------+  +------+  +----------+  |
+------------------------------------------------------------------+
```

**Notification API Service** သည် notification request ကို intake, validation & enrichment ပြုလုပ်ပြီး Kafka publish လုပ်သည်။

**Template Service** သည် template CRUD, variable substitution (Jinja2/Handlebars), multi-language (i18n) support, A/B testing templates တို့ကို manage လုပ်သည်။

**Preference Service** သည် user channel preferences, DND schedule management, global/per-service opt-out တို့ကို ကိုင်တွယ်သည်။

**Channel Adapters** သည် provider-specific SDK wrapping, rate limit management, error handling & normalization ပြုလုပ်သည်။

**Delivery Tracker Service** သည် delivery status updates (sent/delivered/read/failed), retry queue management, analytics event publishing တို့ကို track လုပ်သည်။

---

## ၄၆.၃ Event-Driven Pipeline (Kafka -> Worker -> Channel)

### Overall Pipeline

```
External Services                     Notification Platform
(Uber, Facebook, Bank)

+---------------+   HTTP POST    +------------------+
|  Trip Service | ------------> | Notification API  |
+---------------+               +------------------+
                                         |
+---------------+   HTTP POST            | Validate & Enrich
|  Auth Service | ------------>  +-------+---------+
+---------------+                | Kafka Topic:     |
                                 | notifications    |
+---------------+   HTTP POST   +-------+---------+
| Payment Svc   | ------------>          |
+---------------+                        |
                        +---------------+-+---------------+
                        |               |                 |
                        v               v                 v
               +----------+    +----------+    +----------+
               | Push     |    |  Email   |    |   SMS    |
               | Worker   |    |  Worker  |    |  Worker  |
               +----------+    +----------+    +----------+
                    |               |                |
                    v               v                v
               +-------+    +----------+    +--------+
               |FCM/   |    | SendGrid |    | Twilio |
               |APNS   |    |          |    |        |
               +-------+    +----------+    +--------+
                    |               |                |
                    +-------Delivery Status---------+
                                    |
                           +------------------+
                           | Delivery Tracker |
                           +------------------+
```

### Notification Request Flow

```python
class NotificationAPIService:

    async def send_notification(self, request: NotificationRequest):
        """
        Notification request ကို receive ပြီး pipeline ထဲ ထည့်
        """
        # Step 1: Validate request
        self.validate(request)

        # Step 2: User preferences check
        preferences = await self.preference_service.get(request.user_id)

        eligible_channels = []
        for channel in request.channels:
            if self.is_channel_eligible(channel, preferences, request.priority):
                eligible_channels.append(channel)

        if not eligible_channels:
            return {"status": "suppressed", "reason": "user_preference"}

        # Step 3: Template rendering
        if request.template_id:
            content = await self.template_service.render(
                request.template_id,
                request.template_variables,
                language=preferences.language
            )
        else:
            content = request.content

        # Step 4: Create notification record
        notification_id = await self.db.create({
            "user_id": request.user_id,
            "content": content,
            "channels": eligible_channels,
            "status": "PENDING",
            "priority": request.priority,
            "created_at": datetime.now()
        })

        # Step 5: Kafka publish (per channel)
        for channel in eligible_channels:
            await self.kafka.publish(
                topic=f"notifications.{channel.lower()}",
                key=request.user_id,  # Same user -> same partition (ordering)
                value={
                    "notification_id": notification_id,
                    "user_id": request.user_id,
                    "channel": channel,
                    "content": content,
                    "priority": request.priority
                }
            )

        return {"status": "queued", "notification_id": notification_id}
```

### Channel Worker Implementation

```python
class PushNotificationWorker:

    async def process(self, message):
        """
        Kafka consumer - push notification process
        """
        notification = message.value

        device_tokens = await self.device_service.get_tokens(
            notification["user_id"]
        )

        if not device_tokens:
            await self.mark_failed(notification["notification_id"], "no_device")
            return

        results = []
        for token in device_tokens:
            if token.platform == "android":
                result = await self.fcm_adapter.send(
                    token=token.value,
                    title=notification["content"]["title"],
                    body=notification["content"]["body"],
                    data=notification["content"].get("data", {})
                )
            elif token.platform == "ios":
                result = await self.apns_adapter.send(
                    device_token=token.value,
                    alert=notification["content"]["title"],
                    body=notification["content"]["body"]
                )
            results.append(result)

        success = any(r.success for r in results)
        await self.delivery_tracker.update(
            notification["notification_id"],
            status="DELIVERED" if success else "FAILED",
            provider_response=results
        )
```

---

## ၄၆.၄ Priority Queues (Transactional vs Marketing vs System)

Priority-based routing သည် critical notifications (OTP, payment alert) ကို marketing emails ထက် ဦးစားပေး deliver လုပ်ရန် အရေးကြီးသည်။

```
Priority Levels:
+------------------------------------------+
| CRITICAL  | OTP, Security Alerts         |  -- Immediate, < 1 second
+------------------------------------------+
| HIGH      | Transaction confirmations    |  -- Fast, < 5 seconds
|           | Ride status updates          |
+------------------------------------------+
| MEDIUM    | System notifications         |  -- Normal, < 1 minute
|           | App updates                  |
+------------------------------------------+
| LOW       | Marketing, Promotions        |  -- Batch, < 30 minutes
|           | Newsletters                  |
+------------------------------------------+

Kafka Topics per priority:
  "CRITICAL": "notifications.push.critical"    (10 partitions, 50 workers)
  "HIGH":     "notifications.push.high"         (20 partitions, 30 workers)
  "MEDIUM":   "notifications.push.medium"       (10 partitions, 20 workers)
  "LOW":      "notifications.push.low"          (5 partitions, 10 workers)
```

System load မြင့်သောအခါ low priority notifications ကို throttle ပြုလုပ်သည်:

```python
class PriorityQueueManager:

    def should_throttle_low_priority(self):
        cpu_usage = self.metrics.get_cpu_usage()
        queue_depth_critical = self.metrics.get_queue_depth("CRITICAL")
        # Critical queue backed up -> pause low priority
        return queue_depth_critical > 1000 or cpu_usage > 80
```

---

## ၄၆.၅ Rate Limiting per Channel & User

Different channels တွင် provider-imposed limits ကွဲပြားသည်။ Provider limits နှင့် business limits (user experience protection) နှစ်မျိုးလုံး enforce လုပ်ရသည်။

```python
RATE_LIMITS = {
    # Provider limits
    "FCM":     {"per_project_per_second": 600, "per_device_per_hour": 240},
    "APNS":    {"per_bundle_per_second": 300, "per_device_per_hour": 240},
    "TWILIO":  {"per_account_per_second": 100, "per_number_per_second": 1},
    "SENDGRID":{"per_account_per_hour": 600000},

    # Business limits
    "user_daily_push": 20,
    "user_daily_sms": 5,
    "user_daily_marketing_email": 3
}

class RateLimiter:

    async def check_and_increment(self, key, limit, window_seconds):
        """
        Sliding window rate limiter using Redis sorted sets
        """
        pipe = self.redis.pipeline()
        now = time.time()
        window_start = now - window_seconds

        pipe.zremrangebyscore(key, 0, window_start)
        pipe.zcard(key)
        pipe.zadd(key, {str(now): now})
        pipe.expire(key, window_seconds)

        results = await pipe.execute()
        current_count = results[1]

        return current_count < limit, current_count + 1
```

---

## ၄၆.၆ Delivery Tracking & Retry (Exponential Backoff)

### Retry Architecture

```
Send Notification -> Provider API Call
                           |
              +------------+------------+
              |                         |
          Success                    Failure
              |                         |
     Update status:          +----------+----------+
     "DELIVERED"             |          |          |
                       Transient    Permanent   Unknown
                       Error        Error       Error
                       (5xx, net)   (invalid    (timeout)
                             |       token)          |
                             |          |            |
                        Retry Queue  Mark FAILED  Retry 1x
                             |
                    Exponential Backoff:
                    Attempt 1: 30 seconds
                    Attempt 2: 2 minutes
                    Attempt 3: 8 minutes
                    Attempt 4: 30 minutes
                    Attempt 5: 2 hours
                    After 5 attempts: Dead Letter Queue
```

```python
class RetryStrategy:

    MAX_ATTEMPTS = 5
    BACKOFF_DELAYS = [30, 120, 480, 1800, 7200]  # seconds

    async def handle_delivery_failure(self, notification_id, error, attempt):
        if self.is_permanent_error(error):
            await self.mark_permanently_failed(notification_id, error)
            return

        if attempt >= self.MAX_ATTEMPTS:
            await self.move_to_dlq(notification_id, error)
            return

        delay = self.BACKOFF_DELAYS[attempt] * (1 + random.uniform(0, 0.1))  # Jitter
        retry_at = datetime.now() + timedelta(seconds=delay)

        await self.retry_queue.schedule(
            notification_id=notification_id,
            attempt=attempt + 1,
            execute_at=retry_at
        )

    def is_permanent_error(self, error):
        PERMANENT = ["InvalidRegistration", "NotRegistered",
                     "BadDeviceToken", "Unsubscribed"]
        return error.code in PERMANENT
```

---

## ၄၆.၇ User Preference & Do-Not-Disturb

```python
class PreferenceService:

    def is_channel_eligible(self, channel, preferences, priority):
        """
        Channel ကို send လုပ်ရမည်/မလုပ်ရ ဆုံးဖြတ်
        """
        # Critical priority -> always send (DND bypass)
        if priority == "CRITICAL":
            return True

        if channel in preferences.opted_out_channels:
            return False

        if self.is_dnd_active(preferences.dnd_schedule):
            return priority in ["CRITICAL", "HIGH"]

        return True

    def is_dnd_active(self, dnd_schedule):
        """
        Do-Not-Disturb active status စစ်ဆေးခြင်း
        User timezone ကို respect ပြုသည်
        """
        if not dnd_schedule or not dnd_schedule.enabled:
            return False

        now = datetime.now(pytz.timezone(dnd_schedule.timezone))
        current_time = now.time()

        start = time.fromisoformat(dnd_schedule.start_time)  # "22:00"
        end = time.fromisoformat(dnd_schedule.end_time)      # "07:00"

        # Overnight schedule (22:00 - 07:00)
        if start > end:
            return current_time >= start or current_time <= end
        else:
            return start <= current_time <= end
```

---

## ၄၆.၈ Architecture Diagram

```
+--------------------------------------------------------------------+
|                     Notification Platform                          |
+--------------------------------------------------------------------+
                                |
         External Services      |
         +----------+           |
         | Services | --HTTP--> | Notification API
         +----------+           |      |
                                |      v
                                | +-----------+  +------------------+
                                | | Template  |  | Preference       |
                                | | Service   |  | Service          |
                                | +-----------+  +------------------+
                                |      |
                                | +----+----+-----+-------+
                                | |    |    |     |       |
                                | v    v    v     v       v
                                | Kafka Topics (per channel + priority)
                                | |    |    |     |       |
                                | v    v    v     v       v
                                | Push SMS Email In-App WhatsApp
                                | Workers
                                |  |   |    |     |       |
                                | FCM TWILIO SendG WS    WA-API
                                |  |   |    |     |       |
                                +--+---+----+-----+-------+
                                              |
                                    Delivery Status Updates
                                              |
                                    +------------------+
                                    | Delivery Tracker |
                                    +------------------+
                                              |
                                    +------------------+
                                    | Retry Queue      |
                                    | (Redis Sorted    |
                                    |  Set)            |
                                    +------------------+
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **Channel Abstraction:** FCM, APNS, Twilio, SendGrid ကို adapter pattern ဖြင့် wrap လုပ်ခြင်းသည် business logic ကို provider implementation မှ isolate ပြုသည်။ New channels ထည့်ရာတွင် adapter တစ်ခုသာ implement လုပ်ရသည်။

2. **Priority-Based Routing:** Transactional notifications (OTP, payment) နှင့် marketing notifications ကို separate Kafka topics ဖြင့် partition ပြုခြင်းသည် critical notifications ကို always timely delivery ဖြစ်စေသည်။

3. **Exponential Backoff with Jitter:** Retry storms (thundering herd) ကို ကာကွယ်ရန် jitter ပေါင်းထည့်သော exponential backoff ကို implement လုပ်ရသည်။

4. **Permanent vs Transient Errors:** Invalid device tokens (permanent) ကို immediately DLQ သို့ ပို့ပြီး network errors (transient) ကိုသာ retry လုပ်ရသည်။ ဤ distinction သည် unnecessary retries ကို 60%+ လျှော့ချနိုင်သည်။

5. **User Preference & DND:** CRITICAL priority notifications သည် DND ကို bypass နိုင်ပြီး LOW priority notifications ကို DND hours တွင် suppress ပြုရသည်။ User experience ကို respect ပြုခြင်းသည် long-term engagement ကို တိုးမြှင့်သည်။

6. **Rate Limiting:** Provider limits (FCM per project) နှင့် business limits (user daily push count) နှစ်မျိုးစလုံး enforce လုပ်ရသည်။ Redis sliding window counter သည် distributed rate limiting အတွက် efficient ဖြစ်သည်။
