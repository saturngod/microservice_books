# အခန်း ၃၅ — Food Delivery System (DoorDash / Uber Eats ကဲ့သို့)

## နိဒါန်း

Food Delivery system သည် real-time logistics, geospatial computing, dynamic pricing, နှင့် multi-party coordination (customer, restaurant, driver) ကို simultaneously handle ပြုလုပ်ရသော complex distributed system ဖြစ်သည်။ DoorDash သည် 37 million monthly active users ကို ဝန်ဆောင်မှုပေးပြီး Uber Eats သည် 6,000+ cities ၌ ဆောင်ရွက်နေသည်။ ဤအခန်းတွင် restaurant discovery, order saga, real-time driver dispatch, ETA prediction, surge pricing, driver earnings, restaurant analytics (CQRS), နှင့် architecture diagram တို့ကို production-grade perspective မှ လေ့လာမည်ဖြစ်သည်။

---

## ₃₅.₁ Requirements

### Functional Requirements

- Restaurant browse/search (location, cuisine, rating)
- Menu view + item customization
- Order place + real-time status tracking
- Driver dispatch + real-time location tracking
- ETA prediction (order ready + delivery)
- Surge pricing (demand/supply based)
- Driver earnings + payout
- Restaurant analytics dashboard
- Rating/review system (restaurant + driver)

### Non-Functional Requirements

| Metric | Target |
|---|---|
| Peak Orders | 10,000 orders/minute |
| Restaurant Count | 500,000+ |
| Active Drivers (peak) | 200,000 concurrent |
| Order-to-Delivery | < 45 minutes (metro) |
| Driver Location Update | Every 4 seconds |
| Dispatch Latency | < 30 seconds |
| ETA Accuracy | +/- 5 minutes |
| System Availability | 99.95% |

### Scale Estimation

```
Order throughput:
  10K orders/min = 167 orders/sec (peak)
  Normal: ~80 orders/sec

Driver location updates:
  200K active drivers * 1 update/4sec = 50,000 updates/sec
  GPS data: ~100 bytes per update
  50,000 * 100 = 5 MB/sec raw location data

Geospatial queries:
  Each order dispatch: scan drivers within 5km radius
  167 dispatches/sec * avg 50 candidate drivers = 8,350 driver lookups/sec

Restaurant search:
  30M DAU * 5 searches/day = 150M searches/day
  150M / 86,400 = ~1,740 searches/sec
  Peak: 3x = ~5,200 searches/sec
```

---

## ₃₅.₂ Domain Decomposition

```
Food Delivery Domain Decomposition:
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway                              │
└──┬────┬────┬────┬────┬────┬────┬────┬───────────────────────┘
   │    │    │    │    │    │    │    │
┌──▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──────────┐
│Rest ││Menu││Ordr││Disp││Drvr││Dlvr││Noti││Analytics │
│ Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││  Svc     │
└─────┘└────┘└────┘└────┘└────┘└────┘└────┘└──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Pricing Svc      │
                    │ (Surge + Fees)     │
                    └────────────────────┘
```

| Service | Responsibility | Database |
|---|---|---|
| **Restaurant Service** | Restaurant CRUD, hours, location | PostgreSQL + Elasticsearch |
| **Menu Service** | Menu items, customization, availability | PostgreSQL + Redis cache |
| **Order Service** | Order lifecycle, saga orchestration | PostgreSQL |
| **Dispatch Service** | Driver-order matching, geospatial | Redis Geo + H3 index |
| **Driver Service** | Driver profile, status, location | Redis (real-time) + PostgreSQL |
| **Delivery Service** | Delivery tracking, proof-of-delivery | PostgreSQL + Redis |
| **Notification Service** | Push, SMS, in-app notifications | Redis queue + FCM/APNs |
| **Analytics Service** | Restaurant/driver analytics (CQRS) | ClickHouse (OLAP) |
| **Pricing Service** | Surge pricing, delivery fees, promos | Redis + rules engine |

---

## ₃₅.₃ Order Saga

```
Food Delivery Order Saga:
┌────────────────────────────────────────────────────────────────────┐
│                    ORDER SAGA ORCHESTRATOR                        │
│                                                                    │
│  ┌─────────┐   ┌────────────┐   ┌──────────┐   ┌──────────┐    │
│  │  Place  │──>│ Restaurant │──>│  Assign  │──>│  Pick Up │    │
│  │  Order  │   │   Accept   │   │  Driver  │   │          │    │
│  └─────────┘   └────────────┘   └──────────┘   └──────────┘    │
│       │              │               │               │           │
│       │              │               │          ┌────▼─────┐    │
│       │              │               │          │ Deliver  │    │
│       │              │               │          │          │    │
│       │              │               │          └──────────┘    │
│                                                                    │
│  Compensation:                                                     │
│  Restaurant rejects → Refund → Cancel → Notify Customer           │
│  No driver found → Retry (3x) → Escalate → Manual assign/cancel  │
│  Driver cancels → Reassign driver → Update ETA                    │
└────────────────────────────────────────────────────────────────────┘
```

```python
# Order Saga — Food Delivery specific
class FoodDeliveryOrderSaga:
    """Food delivery order lifecycle manage ပြုလုပ်သည်"""

    async def execute(self, order):
        saga_log = []

        try:
            # Step 1: Payment hold (pre-authorize, capture later)
            payment_hold = await self.payment.pre_authorize(
                user_id=order.customer_id,
                amount=order.total_with_fees,
                idempotency_key=f"order-{order.id}"
            )
            saga_log.append(("payment_hold", payment_hold.id))

            # Step 2: Restaurant ကို order notify + acceptance wait
            restaurant_response = await self.restaurant.send_order(
                restaurant_id=order.restaurant_id,
                items=order.items,
                order_id=order.id,
                timeout=120  # Restaurant 2 min အတွင်း accept ပြုလုပ်ရမည်
            )

            if not restaurant_response.accepted:
                raise RestaurantRejectError(restaurant_response.reason)

            saga_log.append(("restaurant_accepted", order.restaurant_id))
            # Restaurant ၏ estimated prep time update
            order.prep_time = restaurant_response.estimated_prep_minutes

            # Step 3: Driver dispatch — optimal driver ရှာသည်
            # Prep time ၏ 70% ကြာသောအခါ driver assign (timing optimization)
            dispatch_delay = order.prep_time * 0.7 * 60  # seconds
            driver_assignment = await self.dispatch.assign_driver(
                order_id=order.id,
                pickup_location=order.restaurant_location,
                dropoff_location=order.customer_location,
                scheduled_pickup=datetime.utcnow() + timedelta(seconds=dispatch_delay),
                max_retries=3
            )

            if not driver_assignment:
                raise NoDriverAvailableError("No driver found after 3 attempts")

            saga_log.append(("driver_assigned", driver_assignment.driver_id))

            # Step 4: ETA calculate + customer notify
            eta = await self.eta_service.predict(
                restaurant_location=order.restaurant_location,
                customer_location=order.customer_location,
                prep_time=order.prep_time,
                driver_location=driver_assignment.driver_location
            )
            order.estimated_delivery = eta

            await self.notification.send(order.customer_id, "order_confirmed", {
                "order_id": order.id,
                "eta": eta.isoformat(),
                "driver_name": driver_assignment.driver_name,
            })

            return order

        except RestaurantRejectError:
            await self._compensate(order, saga_log, "Restaurant rejected order")
            raise
        except NoDriverAvailableError:
            await self._compensate(order, saga_log, "No driver available")
            raise

    async def _compensate(self, order, saga_log, reason):
        """Compensation — reverse saga steps"""
        for step, step_id in reversed(saga_log):
            if step == "driver_assigned":
                await self.dispatch.unassign_driver(step_id, order.id)
            elif step == "restaurant_accepted":
                await self.restaurant.cancel_order(order.id, reason)
            elif step == "payment_hold":
                await self.payment.release_hold(step_id)

        await self.notification.send(order.customer_id, "order_cancelled", {
            "order_id": order.id,
            "reason": reason
        })
```

---

## ₃₅.₄ Real-Time Driver Dispatch

### Geospatial Index — Redis Geo + H3

```python
# Driver dispatch — geospatial matching
import h3
import redis

class DispatchService:
    """Real-time driver dispatch — geospatial + scoring algorithm"""

    def __init__(self, redis_client):
        self.redis = redis_client
        # H3 resolution 7 — hexagon ~5.16 km2 area
        self.h3_resolution = 7

    def update_driver_location(self, driver_id, lat, lng):
        """Driver GPS location update — 4 seconds interval"""
        # Redis Geo — O(log(N)) insert
        self.redis.geoadd("drivers:active", lng, lat, driver_id)

        # H3 hexagonal index — zone-based lookup
        h3_index = h3.latlng_to_cell(lat, lng, self.h3_resolution)
        # Old zone cleanup + new zone add
        pipe = self.redis.pipeline()
        pipe.srem(f"drivers:zone:*", driver_id)  # Simplified
        pipe.sadd(f"drivers:zone:{h3_index}", driver_id)
        pipe.hset(f"driver:{driver_id}", mapping={
            "lat": str(lat),
            "lng": str(lng),
            "updated_at": str(time.time()),
            "h3_zone": h3_index,
        })
        pipe.execute()

    async def assign_driver(self, order_id, pickup_location, dropoff_location,
                            scheduled_pickup, max_retries=3):
        """Order အတွက် optimal driver ရှာပြီး assign ပြုလုပ်သည်"""

        for attempt in range(max_retries):
            # Step 1: Nearby drivers ရှာသည် — pickup location ၏ 5km radius
            nearby_drivers = self.redis.georadius(
                "drivers:active",
                pickup_location.lng,
                pickup_location.lat,
                5,                    # 5 km radius
                unit="km",
                withcoord=True,
                withdist=True,
                sort="ASC",           # Closest first
                count=20              # Top 20 candidates
            )

            if not nearby_drivers:
                # Radius ချဲ့ပြီး ထပ်ရှာ
                nearby_drivers = self.redis.georadius(
                    "drivers:active",
                    pickup_location.lng, pickup_location.lat,
                    10, unit="km", withcoord=True, withdist=True,
                    sort="ASC", count=20
                )

            # Step 2: Available drivers filter (busy/offline exclude)
            candidates = []
            for driver_id, dist, (lng, lat) in nearby_drivers:
                driver_status = self.redis.hget(f"driver:{driver_id.decode()}", "status")
                if driver_status == b"available":
                    candidates.append({
                        "driver_id": driver_id.decode(),
                        "distance_km": dist,
                        "lat": lat,
                        "lng": lng,
                    })

            if not candidates:
                await asyncio.sleep(10)  # Wait and retry
                continue

            # Step 3: Scoring — optimal driver ရွေးချယ်
            scored = self._score_candidates(
                candidates, pickup_location, dropoff_location
            )

            # Step 4: Driver offer — accept/decline wait
            for candidate in scored:
                accepted = await self._offer_to_driver(
                    candidate["driver_id"], order_id, timeout=30
                )
                if accepted:
                    # Driver status update
                    self.redis.hset(
                        f"driver:{candidate['driver_id']}",
                        "status", "assigned"
                    )
                    return DriverAssignment(
                        driver_id=candidate["driver_id"],
                        driver_location=(candidate["lat"], candidate["lng"]),
                        estimated_pickup_time=candidate["eta_to_restaurant"],
                    )

            # All drivers declined — retry
            await asyncio.sleep(5)

        return None  # No driver found after max_retries

    def _score_candidates(self, candidates, pickup, dropoff):
        """Driver scoring — distance, rating, acceptance rate ပေါင်းစပ်"""
        scored = []
        for c in candidates:
            driver_stats = self.redis.hgetall(f"driver:{c['driver_id']}")
            rating = float(driver_stats.get(b"rating", 4.5))
            acceptance_rate = float(driver_stats.get(b"acceptance_rate", 0.8))

            # Score calculation — lower is better
            score = (
                c["distance_km"] * 0.4 +             # Distance weight
                (5.0 - rating) * 0.3 +                # Rating weight (inverted)
                (1.0 - acceptance_rate) * 10 * 0.3    # Acceptance rate weight
            )

            c["score"] = score
            c["eta_to_restaurant"] = c["distance_km"] / 30 * 60  # minutes (30km/h avg)
            scored.append(c)

        return sorted(scored, key=lambda x: x["score"])
```

```
Driver Dispatch Flow:
┌────────────┐    ┌──────────────┐    ┌──────────────┐
│   Order    │───>│   Dispatch   │───>│  Redis Geo   │
│  Created   │    │   Service    │    │  GEORADIUS    │
└────────────┘    └──────┬───────┘    └──────────────┘
                         │
                    ┌────▼─────────┐
                    │  Score &     │
                    │  Rank        │
                    │  Candidates  │
                    └────┬─────────┘
                         │
                    ┌────▼─────────┐    ┌─────────────┐
                    │  Offer to    │───>│   Driver    │
                    │  Top Driver  │    │   App       │
                    └────┬─────────┘    └──────┬──────┘
                         │                      │
                    ┌────▼─────────┐    Accept/Decline
                    │  Assignment  │◄───────────┘
                    │  Confirmed   │
                    └──────────────┘
```

---

## ₃₅.₅ ETA Prediction

```python
# ETA prediction — ML model + live traffic signals
class ETAPredictor:
    """Delivery ETA predict ပြုလုပ်သည်
    ML model + live traffic + restaurant prep time"""

    def predict(self, restaurant_location, customer_location,
                prep_time_minutes, driver_location, weather=None):
        """Total delivery ETA predict"""

        # Component 1: Driver → Restaurant ETA
        driver_to_restaurant = self._predict_travel_time(
            origin=driver_location,
            destination=restaurant_location,
            mode="driving"
        )

        # Component 2: Restaurant prep time (from restaurant estimate)
        # Historical data ဖြင့် adjust — restaurant's actual vs estimated
        adjusted_prep = self._adjust_prep_time(
            restaurant_id=restaurant_location.restaurant_id,
            estimated_prep=prep_time_minutes
        )

        # Component 3: Restaurant → Customer ETA
        restaurant_to_customer = self._predict_travel_time(
            origin=restaurant_location,
            destination=customer_location,
            mode="driving"
        )

        # Component 4: Buffer time (parking, stairs, handoff)
        buffer = self._calculate_buffer(
            customer_location,
            time_of_day=datetime.now().hour
        )

        # Total ETA
        total_minutes = (
            driver_to_restaurant +
            max(0, adjusted_prep - driver_to_restaurant) +  # Overlap
            restaurant_to_customer +
            buffer
        )

        return {
            "total_minutes": round(total_minutes),
            "breakdown": {
                "driver_to_restaurant": round(driver_to_restaurant),
                "prep_time": round(adjusted_prep),
                "restaurant_to_customer": round(restaurant_to_customer),
                "buffer": round(buffer),
            },
            "confidence": self._calculate_confidence(total_minutes),
        }

    def _predict_travel_time(self, origin, destination, mode):
        """ML model ဖြင့် travel time predict
        Features: distance, time_of_day, day_of_week, weather, traffic"""
        features = {
            "distance_km": self._haversine(origin, destination),
            "hour_of_day": datetime.now().hour,
            "day_of_week": datetime.now().weekday(),
            "is_rush_hour": 1 if datetime.now().hour in [7,8,9,17,18,19] else 0,
            "weather_condition": self._get_weather(origin),
            "live_traffic_factor": self._get_traffic_factor(origin, destination),
        }

        # ML model inference — pre-trained gradient boosting model
        predicted_minutes = self.ml_model.predict([list(features.values())])[0]
        return max(2, predicted_minutes)  # Minimum 2 minutes

    def _adjust_prep_time(self, restaurant_id, estimated_prep):
        """Restaurant ၏ historical prep time data ဖြင့် estimate adjust"""
        # Recent 100 orders ၏ actual vs estimated ratio
        actual_ratio = self.redis.get(f"restaurant:{restaurant_id}:prep_ratio")
        if actual_ratio:
            ratio = float(actual_ratio)
            # Restaurant estimate * historical accuracy ratio
            return estimated_prep * ratio
        return estimated_prep
```

---

## ₃₅.₆ Surge Pricing Engine

```python
# Surge pricing — H3 zone based demand/supply ratio
class SurgePricingEngine:
    """Zone-based surge pricing — demand/supply ratio"""

    def __init__(self, redis_client):
        self.redis = redis_client
        self.h3_resolution = 7  # ~5km2 hexagons

    def calculate_surge_multiplier(self, location):
        """Location ၏ surge multiplier calculate"""
        h3_index = h3.latlng_to_cell(location.lat, location.lng, self.h3_resolution)

        # Demand: zone ၏ recent orders count (last 15 min)
        demand = int(self.redis.get(f"demand:{h3_index}:15min") or 0)

        # Supply: zone ၏ available drivers count
        supply = int(self.redis.scard(f"drivers:zone:{h3_index}") or 0)

        if supply == 0:
            return 2.5  # Maximum surge — no drivers

        demand_supply_ratio = demand / max(supply, 1)

        # Surge tiers — ratio ပေါ်မူတည်ပြီး multiplier
        if demand_supply_ratio < 1.0:
            return 1.0       # No surge — supply adequate
        elif demand_supply_ratio < 1.5:
            return 1.2       # Light surge
        elif demand_supply_ratio < 2.0:
            return 1.5       # Moderate surge
        elif demand_supply_ratio < 3.0:
            return 1.8       # Heavy surge
        else:
            return min(2.5, 1.0 + demand_supply_ratio * 0.5)  # Cap at 2.5x

    def calculate_delivery_fee(self, restaurant_location, customer_location):
        """Delivery fee = base + distance + surge"""
        distance_km = self._haversine(restaurant_location, customer_location)
        surge = self.calculate_surge_multiplier(customer_location)

        base_fee = 2.99
        distance_fee = distance_km * 0.50  # $0.50 per km
        surge_fee = (base_fee + distance_fee) * (surge - 1)

        total = base_fee + distance_fee + surge_fee
        return {
            "base_fee": round(base_fee, 2),
            "distance_fee": round(distance_fee, 2),
            "surge_multiplier": surge,
            "surge_fee": round(surge_fee, 2),
            "total_delivery_fee": round(total, 2),
        }

    def record_demand(self, location):
        """Order request ဖြစ်သောအခါ demand counter increment"""
        h3_index = h3.latlng_to_cell(location.lat, location.lng, self.h3_resolution)
        key = f"demand:{h3_index}:15min"
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.expire(key, 900)  # 15 min TTL
        pipe.execute()
```

```
Surge Pricing Zones (H3 Hexagonal Grid):
┌─────┬─────┬─────┬─────┐
│     │     │ 1.5x│     │  ← Moderate surge zone
│ 1.0 │ 1.0 │surge│ 1.0 │
│     │     │     │     │
├─────┼─────┼─────┼─────┤
│     │ 2.0x│ 2.5x│ 1.8x│  ← High demand area
│ 1.0 │surge│surge│surge│     (dinner rush + rain)
│     │     │     │     │
├─────┼─────┼─────┼─────┤
│     │ 1.2x│ 1.5x│     │
│ 1.0 │surge│surge│ 1.0 │
│     │     │     │     │
└─────┴─────┴─────┴─────┘

Each hexagon = H3 resolution 7 (~5km2)
Demand/Supply ratio → surge multiplier
Updated every 15 seconds
```

---

## ₃₅.₇ Driver Earnings & Payout

```python
# Driver earnings ledger — double-entry accounting
class DriverEarningsService:
    """Driver earnings manage — ledger-based system"""

    def record_delivery_earning(self, driver_id, order_id, delivery_details):
        """Delivery complete ပြီးနောက် earnings record"""
        base_pay = 3.50  # Base delivery pay
        distance_pay = delivery_details.distance_km * 0.60  # Per km
        time_pay = delivery_details.actual_minutes * 0.15   # Per minute
        tip = delivery_details.customer_tip or 0
        surge_bonus = delivery_details.surge_multiplier * 1.50 if delivery_details.surge_multiplier > 1 else 0

        total_earning = base_pay + distance_pay + time_pay + tip + surge_bonus

        # Ledger entry — auditability
        earning = DriverEarning(
            driver_id=driver_id,
            order_id=order_id,
            base_pay=round(base_pay, 2),
            distance_pay=round(distance_pay, 2),
            time_pay=round(time_pay, 2),
            tip=round(tip, 2),
            surge_bonus=round(surge_bonus, 2),
            total=round(total_earning, 2),
            timestamp=datetime.utcnow(),
        )

        # Database save + real-time balance update
        self.db.save(earning)
        self.redis.incrbyfloat(f"driver:{driver_id}:balance", total_earning)

        return earning

    def process_weekly_payout(self, driver_id):
        """Weekly payout — driver bank account သို့ transfer"""
        # Balance calculate — week ၏ earnings sum
        weekly_earnings = self.db.query(
            "SELECT SUM(total) FROM driver_earnings "
            "WHERE driver_id = %s AND timestamp >= %s AND paid = false",
            [driver_id, self._week_start()]
        )

        if weekly_earnings <= 0:
            return None

        # Platform commission deduct (20%)
        commission = weekly_earnings * 0.20
        payout_amount = weekly_earnings - commission

        # Bank transfer initiate
        transfer = self.payment_provider.transfer(
            driver_id=driver_id,
            amount=payout_amount,
            currency="USD",
            idempotency_key=f"payout-{driver_id}-{self._week_start()}"
        )

        # Mark earnings as paid
        self.db.execute(
            "UPDATE driver_earnings SET paid = true, payout_id = %s "
            "WHERE driver_id = %s AND timestamp >= %s AND paid = false",
            [transfer.id, driver_id, self._week_start()]
        )

        return {
            "gross_earnings": round(weekly_earnings, 2),
            "commission": round(commission, 2),
            "payout": round(payout_amount, 2),
            "transfer_id": transfer.id,
        }
```

---

## ₃₅.₈ Restaurant Analytics (CQRS)

```
CQRS Pattern for Restaurant Analytics:
┌──────────────────────────────────────────────────────────────┐
│  Command Side (Write)              Query Side (Read)         │
│                                                              │
│  ┌──────────┐                      ┌──────────────┐         │
│  │  Order   │──event──>Kafka──────>│  Analytics   │         │
│  │  Service │                      │  Consumer    │         │
│  └──────────┘                      └──────┬───────┘         │
│  ┌──────────┐                             │                  │
│  │  Driver  │──event──>Kafka──────────────>│                  │
│  │  Service │                             │                  │
│  └──────────┘                      ┌──────▼───────┐         │
│  ┌──────────┐                      │  ClickHouse  │         │
│  │  Rating  │──event──>Kafka──────>│  (OLAP DB)   │         │
│  │  Service │                      └──────┬───────┘         │
│  └──────────┘                             │                  │
│                                    ┌──────▼───────┐         │
│                                    │  Restaurant  │         │
│                                    │  Dashboard   │         │
│                                    │  (Read API)  │         │
│                                    └──────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

```python
# Restaurant analytics — ClickHouse queries
class RestaurantAnalyticsService:
    """CQRS read side — ClickHouse ClickHouse ဖြင့် fast analytics"""

    def __init__(self, clickhouse_client):
        self.ch = clickhouse_client

    def get_dashboard(self, restaurant_id, date_range):
        """Restaurant owner dashboard data"""

        # Revenue summary
        revenue = self.ch.execute("""
            SELECT
                toDate(order_time) as date,
                count() as order_count,
                sum(order_total) as revenue,
                avg(order_total) as avg_order_value,
                countIf(status = 'cancelled') as cancellations
            FROM orders
            WHERE restaurant_id = %(rid)s
              AND order_time BETWEEN %(start)s AND %(end)s
            GROUP BY date
            ORDER BY date
        """, {"rid": restaurant_id, "start": date_range.start, "end": date_range.end})

        # Popular items
        popular_items = self.ch.execute("""
            SELECT
                item_name,
                count() as order_count,
                sum(item_price * quantity) as revenue
            FROM order_items
            WHERE restaurant_id = %(rid)s
              AND order_time BETWEEN %(start)s AND %(end)s
            GROUP BY item_name
            ORDER BY order_count DESC
            LIMIT 10
        """, {"rid": restaurant_id, "start": date_range.start, "end": date_range.end})

        # Prep time accuracy
        prep_accuracy = self.ch.execute("""
            SELECT
                avg(actual_prep_minutes) as avg_actual,
                avg(estimated_prep_minutes) as avg_estimated,
                avg(actual_prep_minutes - estimated_prep_minutes) as avg_deviation
            FROM deliveries
            WHERE restaurant_id = %(rid)s
              AND delivery_time BETWEEN %(start)s AND %(end)s
        """, {"rid": restaurant_id, "start": date_range.start, "end": date_range.end})

        # Rating trend
        ratings = self.ch.execute("""
            SELECT
                toDate(created_at) as date,
                avg(rating) as avg_rating,
                count() as review_count
            FROM reviews
            WHERE restaurant_id = %(rid)s
              AND created_at BETWEEN %(start)s AND %(end)s
            GROUP BY date
            ORDER BY date
        """, {"rid": restaurant_id, "start": date_range.start, "end": date_range.end})

        return {
            "revenue": revenue,
            "popular_items": popular_items,
            "prep_accuracy": prep_accuracy,
            "ratings": ratings,
        }
```

---

## ₃₅.₉ Architecture Diagram

```
Food Delivery System — Full Architecture:
┌─────────────────────────────────────────────────────────────────────┐
│                          CDN + WAF                                  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│                      API Gateway                                    │
│         Auth │ Rate Limit │ BFF (Mobile/Web)                       │
└──┬────┬────┬────┬────┬────┬────┬────┬──────────────────────────────┘
   │    │    │    │    │    │    │    │
┌──▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼───────┐
│Rest ││Menu││Ordr││Disp││Drvr││Dlvr││Noti││Pricing│
│ Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││  Svc  │
└──┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└───┬───┘
   │     │     │     │     │     │     │       │
┌──▼──┐┌─▼──┐┌─▼──┐┌─▼────┐┌─▼──┐┌─▼──┐┌─▼──┐┌──▼───┐
│PgSQL││PgSQL││PgSQL││Redis ││Redis││PgSQL││Redis││Redis │
│+ES  ││+Rds ││     ││Geo  ││+PgSQ││+Rds ││Queue││Rules │
└─────┘└─────┘└─────┘└──────┘└─────┘└─────┘└─────┘└──────┘
                │
          ┌─────▼──────────────────────────┐
          │          Kafka Event Bus        │
          │  order.*, driver.*, delivery.* │
          └─────────────┬──────────────────┘
                        │
                  ┌─────▼────────┐
                  │  ClickHouse  │
                  │  (Analytics) │
                  │  CQRS Read   │
                  └──────────────┘

Real-Time Communication:
  Customer App ←──WebSocket──→ Delivery Service (order tracking)
  Driver App ←──WebSocket──→ Dispatch Service (new order offers)
  Driver App ──GPS──→ Driver Service (location updates every 4s)
```

---

## Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Driver Location** | Redis Geo + H3 hexagons | O(1) lookup + consistent zones |
| **Dispatch** | Score-based + batch optimization | Fairness + efficiency |
| **ETA** | ML model + live traffic | Accuracy > simple distance |
| **Surge Pricing** | H3 zone + demand/supply ratio | Granular, responsive |
| **Analytics** | CQRS + ClickHouse | Decouple read/write, fast OLAP |
| **Earnings** | Ledger-based | Auditability + correctness |
| **Order Saga** | Orchestration | Central coordinator, easier debugging |

---

## အဓိက အချက်များ (Key Takeaways)

1. **Real-time driver dispatch** — Redis Geo + H3 hexagonal grid ဖြင့် nearby drivers ကို milliseconds ထဲတွင် ရှာပြီး scoring algorithm ဖြင့် optimal match ပြုလုပ်သည်

2. **Order saga** — restaurant accept → driver assign → pickup → deliver flow ကို compensation logic ဖြင့် reliable orchestrate ပြုလုပ်သည်

3. **ETA prediction** — ML model + live traffic + restaurant historical prep time ဖြင့် accurate delivery estimate ပေးသည်

4. **Surge pricing** — H3 zone-based demand/supply ratio ဖြင့် granular pricing ပြုလုပ်ပြီး driver incentive + demand management achieve ပြုလုပ်သည်

5. **CQRS** — restaurant analytics ကို ClickHouse (OLAP) ဖြင့် serve ပြုလုပ်ပြီး order processing performance ကို impact မဖြစ်စေ

6. **Driver earnings** — ledger-based system ဖြင့် auditability + correctness guarantee ပြုလုပ်ပြီး weekly payout ကို automated process ပြုလုပ်သည်

---

> **နောက်အခန်း Preview:** Part 10 တွင် Real-Time Chat System, Video Streaming Platform တို့ကို ဆွေးနွေးသွားမည်ဖြစ်သည်။
