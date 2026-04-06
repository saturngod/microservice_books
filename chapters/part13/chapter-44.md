# အခန်း ၄၄: Ride-Sharing စနစ် (Uber / Lyft ပုံစံ)

## နိဒါန်း

Ride-sharing platform များသည် modern distributed systems engineering ၏ အကောင်းဆုံး ဥပမာများထဲမှ တစ်ခုဖြစ်သည်။ Uber နှင့် Lyft တို့သည် real-time location tracking, dynamic pricing, complex matching algorithms နှင့် large-scale payment processing တို့ကို တပြိုင်နက် ကိုင်တွယ်ရသော ရှုပ်ထွေးသောစနစ်များဖြစ်သည်။ ဤအခန်းတွင် ride-sharing system တစ်ခုကို microservices architecture ဖြင့် မည်သို့ဒီဇိုင်းဆွဲမည်ဆိုသည်ကို အသေးစိတ် လေ့လာသွားမည်ဖြစ်သည်။

Peak hours တွင် Uber သည် တစ်မိနစ်လျှင် requests သန်းနှင့်ချီသော အရေအတွက်ကို ကိုင်တွယ်ရသည်။ Driver နှင့် rider ကို milliseconds အတွင်း match လုပ်ရသည်။ Surge pricing ကို real-time တွင် တွက်ချက်ရသည်။ ဤအတွက် scalable, fault-tolerant microservices architecture လိုအပ်သည်။

---

## ၄၄.၁ Requirements: Matching, Pricing, ETA, Payments, Analytics

### Functional Requirements

**Rider ဘက်မှ:**
- Ride တောင်းဆိုနိုင်ရမည် (pickup location, destination)
- ကြိုတင်ခန့်မှန်းခ (estimated fare) ကို booking မတိုင်မီ မြင်ရမည်
- Driver ၏ real-time location ကို track လုပ်နိုင်ရမည်
- Trip ပြီးသောအခါ payment အလိုအလျောက် ပြုလုပ်ရမည်
- Rating နှင့် review ပေးနိုင်ရမည်

**Driver ဘက်မှ:**
- Online/offline status ပြောင်းနိုင်ရမည်
- Ride request လက်ခံ သို့မဟုတ် ငြင်းဆန်နိုင်ရမည်
- Navigation guidance ရနိုင်ရမည်
- Weekly payout ရနိုင်ရမည်

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| Availability | 99.99% uptime |
| Driver location update latency | < 5 seconds |
| Matching latency | < 30 seconds |
| Payment processing | < 3 seconds |
| System throughput | 10M+ concurrent users |
| Data consistency | Eventual consistency (location), Strong consistency (payment) |

### Capacity Estimation

```
// မြို့ကြီးတစ်ခုအတွက် ခန့်မှန်းချက်
Active drivers per city        = 50,000
Active riders per city         = 200,000
Location updates per driver    = 1 update/4 seconds
Total location updates/sec     = 50,000 / 4 = 12,500 QPS
Peak ride requests/sec         = 1,000 QPS
Total cities                   = 100

// Global ခန့်မှန်းချက်
Global location QPS            = 12,500 × 100 = 1.25M QPS
Global ride requests/sec       = 1,000 × 100  = 100K QPS
```

---

## ၄၄.၂ Domain Decomposition: Services များ

Ride-sharing system ကို အောက်ပါ microservices များအဖြစ် ခွဲခြမ်းနိုင်သည်:

```
+------------------------------------------------------------------+
|                    Ride-Sharing Platform                         |
+------------------------------------------------------------------+
|                                                                  |
|  +------------+  +------------+  +------------+  +-----------+  |
|  |   User     |  |   Driver   |  |  Matching  |  |   Trip    |  |
|  |  Service   |  |  Service   |  |  Service   |  |  Service  |  |
|  +------------+  +------------+  +------------+  +-----------+  |
|                                                                  |
|  +------------+  +------------+  +------------+  +-----------+  |
|  |  Pricing   |  |  Payment   |  | Notification|  | Analytics |  |
|  |  Service   |  |  Service   |  |  Service   |  |  Service  |  |
|  +------------+  +------------+  +------------+  +-----------+  |
|                                                                  |
|  +------------------+  +----------------------------------+      |
|  |  Location Service |  |        Wallet Service           |      |
|  +------------------+  +----------------------------------+      |
+------------------------------------------------------------------+
```

### Service တစ်ခုချင်းစီ၏ တာဝန်

**User Service:**
- Registration, authentication, profile management
- JWT token issuance
- Database: PostgreSQL (user profiles)

**Driver Service:**
- Driver onboarding, document verification, status management
- Vehicle information management
- Database: PostgreSQL

**Location Service:**
- Real-time driver location tracking
- Geospatial indexing (geohash)
- Database: Redis (current location), Cassandra (history)

**Matching Service:**
- Driver-rider proximity matching
- ETA calculation
- Relies on Location Service

**Trip Service:**
- Trip state machine management
- Trip history
- Database: PostgreSQL + Event Store

**Pricing Service:**
- Base fare calculation
- Surge pricing multiplier
- External: ML model for demand prediction

**Payment Service:**
- Payment method management
- Post-trip charge processing
- Integration: Stripe, Braintree

**Notification Service:**
- Push notifications (FCM/APNS)
- SMS notifications
- In-app real-time updates (WebSocket)

**Analytics Service:**
- Trip metrics, revenue reporting
- Driver performance analytics
- Database: Apache Spark + Data Warehouse

**Wallet Service:**
- Driver earnings tracking
- Weekly payout processing
- Database: PostgreSQL (ACID transactions)

---

## ၄၄.၃ Real-Time Driver Location Tracking

Location tracking သည် ride-sharing system ၏ အဓိကအင်္ဂါရပ်ဖြစ်သည်။ Driver app မှ server သို့ location ကို continuous ပေးပို့ရသည်။

### Architecture Flow

```
+-------------+     WebSocket      +------------------+
|  Driver App | -----------------> | WebSocket Gateway |
+-------------+                    +------------------+
                                           |
                                    Location Event
                                           |
                                           v
                                  +------------------+
                                  | Location Service  |
                                  +------------------+
                                    /             \
                               Write              Read
                                /                   \
                    +-----------+           +------------------+
                    |   Redis   |           |  Geospatial Index|
                    | (current) |           |  (Geohash/R-Tree) |
                    +-----------+           +------------------+
                         |
                    Async write
                         |
                    +-----------+
                    | Cassandra |
                    | (history) |
                    +-----------+
```

### Geohash vs Quad-Tree

**Geohash:**
Geohash သည် geographic coordinates (latitude, longitude) ကို alphanumeric string အဖြစ် encode လုပ်သည်။ Precision level မြင့်လာသည်နှင့်အမျှ area သေးငယ်လာသည်။

```
// Geohash precision levels
Precision 1: ~5,000 km × 5,000 km  (ကမ္ဘာ့ region ကြီးများ)
Precision 4: ~40 km × 20 km        (မြို့နယ် level)
Precision 6: ~1.2 km × 0.6 km      (ရပ်ကွက် level)
Precision 8: ~38 m × 19 m          (မြေကွက် level)

// Driver location ကို geohash encode လုပ်သည်
driver_location = {
    driver_id: "drv_123",
    lat: 16.8661,      // ရန်ကုန် latitude
    lng: 96.1951,      // ရန်ကုန် longitude
    geohash: "w3gv8q"  // precision 6
}

// Neighboring geohashes ကို proximity search အတွက် ရှာသည်
neighbors = geohash.neighbors("w3gv8q")
// => ["w3gv8r", "w3gv8x", "w3gv8w", ...]
```

**Quad-Tree:**
Quad-tree သည် geographic space ကို recursively 4 quadrants ခွဲသည်။ Dynamic density adaptation ပိုကောင်းသည်။

```
// Quad-tree node structure
QuadTreeNode {
    boundary: BoundingBox,
    drivers: [],        // ဤ node တွင် ရှိသော drivers
    children: [         // 4 child nodes (NW, NE, SW, SE)
        QuadTreeNode,
        QuadTreeNode,
        QuadTreeNode,
        QuadTreeNode
    ],
    capacity: 10        // Node တစ်ခုတွင် maximum drivers အရေအတွက်
}
```

| Feature | Geohash | Quad-Tree |
|---------|---------|-----------|
| Implementation | ရိုးရှင်း | ရှုပ်ထွေး |
| Range query | ကောင်း | အကောင်းဆုံး |
| Dynamic density | အကန့်အသတ်ရှိ | ကောင်းမွန် |
| Distributed caching | ကောင်း (string key) | ခက်ခဲ |
| Uber's choice | Geohash + S2 | - |

**Recommendation:** Production system တွင် Redis Geo commands (GEOADD, GEORADIUS) သို့မဟုတ် Uber's H3 library ကို အသုံးပြုသင့်သည်။

```python
import redis

r = redis.Redis()

# Driver location ကို Redis geo index သို့ သိမ်းဆည်း
def update_driver_location(driver_id, lat, lng):
    # Redis GEOADD command ဖြင့် location သိမ်း
    r.geoadd("active_drivers", (lng, lat, driver_id))
    
    # TTL သတ်မှတ် (driver offline ဖြစ်သွားလျှင် auto-expire)
    r.expire(f"driver:{driver_id}:online", 30)

# Nearby drivers ရှာဖွေ
def find_nearby_drivers(rider_lat, rider_lng, radius_km=5):
    # GEORADIUS command ဖြင့် 5km အတွင်း drivers ရှာ
    drivers = r.georadius(
        "active_drivers",
        rider_lng, rider_lat,
        radius_km, unit="km",
        withcoord=True,
        withdist=True,
        sort="ASC",  # နီးစပ်သော driver ကို အရင်ပြ
        count=20     # Maximum 20 candidates
    )
    return drivers
```

---

## ၄၄.၄ Driver Matching Algorithm

Proximity (distance) တစ်ခုတည်းဖြင့် match လုပ်သည်မဟုတ်ဘဲ ETA, driver score တို့ပါ ထည့်သွင်းစဉ်းစားသည်။

### Matching Flow

```
Rider Request
     |
     v
+--------------------+
| Find Candidate     |  -- Geospatial search (radius 5km)
| Drivers            |
+--------------------+
     |
     | (Candidate list: 20 drivers)
     v
+--------------------+
| Calculate ETA for  |  -- Routing service API call
| each candidate     |
+--------------------+
     |
     v
+--------------------+
| Score & Rank       |  -- Multi-factor scoring
| Drivers            |
+--------------------+
     |
     v
+--------------------+
| Dispatch Request   |  -- Send to #1 ranked driver
| to Top Driver      |
+--------------------+
     |
     +-- Accept? --> Confirm Match
     |
     +-- Reject / Timeout (15s) --> Try Next Driver
```

### Scoring Formula

```python
def calculate_driver_score(driver, rider_location):
    # ETA score (နည်းလေ ကောင်းလေ)
    eta_minutes = get_eta(driver.location, rider_location)
    eta_score = max(0, 100 - eta_minutes * 5)
    
    # Driver rating score (5-star system)
    rating_score = (driver.rating / 5.0) * 100
    
    # Acceptance rate score
    acceptance_score = driver.acceptance_rate * 100
    
    # Cancellation penalty
    cancellation_penalty = driver.cancellation_rate * 50
    
    # Weighted final score
    final_score = (
        eta_score * 0.5 +           # ETA အရေးအကြီးဆုံး
        rating_score * 0.3 +         # Rating အရေးကြီး
        acceptance_score * 0.2 -     # Acceptance rate
        cancellation_penalty         # Penalty
    )
    
    return final_score

def match_driver(rider_request):
    # Step 1: Candidate drivers ရှာဖွေ
    candidates = find_nearby_drivers(
        rider_request.pickup_lat,
        rider_request.pickup_lng,
        radius_km=5
    )
    
    # Step 2: Score တွက်ချက်
    scored_drivers = [
        (driver, calculate_driver_score(driver, rider_request.pickup))
        for driver in candidates
    ]
    
    # Step 3: Score အမြင့်ဆုံးကို ဦးစားပေး sort
    scored_drivers.sort(key=lambda x: x[1], reverse=True)
    
    # Step 4: Sequential dispatch
    for driver, score in scored_drivers:
        response = dispatch_to_driver(driver, rider_request, timeout=15)
        if response.accepted:
            return driver
    
    return None  # No driver available
```

---

## ၄၄.၅ Surge Pricing Engine

Surge pricing (dynamic pricing) သည် demand-supply imbalance ရှိသောအခါ fare ကို မြင့်တင်သည်။

### Architecture

```
+----------+    Events    +-------+    Stream    +-----------------+
| Driver   | -----------> |       | -----------> | Demand Analyzer |
| App      |              | Kafka |              +-----------------+
+----------+              |       |                      |
                          |       |              +-----------------+
+----------+    Events    |       | -----------> | Supply Analyzer |
| Rider    | -----------> |       |              +-----------------+
| App      |              +-------+                      |
+----------+                                             v
                                              +---------------------+
                                              | Grid-Level Surge    |
                                              | Calculator          |
                                              | (Geohash grid)      |
                                              +---------------------+
                                                         |
                                              +---------------------+
                                              | Redis Cache         |
                                              | (surge multipliers) |
                                              +---------------------+
                                                         |
                                              +---------------------+
                                              | Pricing Service     |
                                              +---------------------+
```

### Surge Calculation Logic

```python
class SurgePricingEngine:
    
    def calculate_surge_multiplier(self, geohash_cell):
        """
        Geohash grid cell တစ်ခုအတွက် surge multiplier တွက်ချက်
        """
        # Last 5 minutes ၏ data ယူ
        window_minutes = 5
        
        # Active rider requests (demand)
        demand = self.get_demand(geohash_cell, window_minutes)
        
        # Available drivers (supply)
        supply = self.get_supply(geohash_cell)
        
        # Demand/Supply ratio
        if supply == 0:
            ratio = float('inf')  # No drivers available
        else:
            ratio = demand / supply
        
        # Surge multiplier ကို ratio မှ တွက်
        if ratio < 1.2:
            multiplier = 1.0   # Normal pricing
        elif ratio < 2.0:
            multiplier = 1.5   # 1.5x surge
        elif ratio < 3.0:
            multiplier = 2.0   # 2.0x surge
        elif ratio < 4.0:
            multiplier = 2.5   # 2.5x surge
        else:
            multiplier = 3.0   # Maximum 3.0x surge
        
        # Cache ထဲ သိမ်းဆည်း (30 seconds TTL)
        self.redis.setex(
            f"surge:{geohash_cell}",
            30,
            multiplier
        )
        
        return multiplier
    
    def get_fare_estimate(self, pickup_geohash, distance_km, duration_min):
        """
        Final fare တွက်ချက်
        """
        # Base fare
        base_fare = 2.0  # USD
        per_km_rate = 1.5
        per_min_rate = 0.25
        
        base_cost = base_fare + (distance_km * per_km_rate) + (duration_min * per_min_rate)
        
        # Surge multiplier
        surge = self.get_cached_surge(pickup_geohash) or 1.0
        
        final_fare = base_cost * surge
        
        return {
            "base_fare": base_cost,
            "surge_multiplier": surge,
            "total_fare": round(final_fare, 2)
        }
```

---

## ၄၄.၆ Trip Lifecycle State Machine

Trip သည် request မှ completion အထိ ခြေလှမ်းအများအပြားဖြင့် ဖြတ်သန်းသည်။

```
REQUEST_PENDING
     |
     | (Driver matched)
     v
DRIVER_ASSIGNED
     |
     | (Driver accepts)
     v
DRIVER_EN_ROUTE       <-- Driver သည် rider ဆီသွားနေဆဲ
     |
     | (Driver arrives at pickup)
     v
DRIVER_ARRIVED        <-- Driver သည် pickup location ရောက်ပြီ
     |
     | (Rider boards)
     v
TRIP_IN_PROGRESS      <-- Trip တကယ်စစ်စစ် ဖြစ်နေဆဲ
     |
     | (Reach destination)
     v
TRIP_COMPLETED        <-- Trip ပြီးဆုံး၊ payment process
     |
CANCELLED             <-- မည်သည့် state မှမဆို cancel ဖြစ်နိုင်
```

### State Machine Implementation

```python
from enum import Enum
from dataclasses import dataclass
from datetime import datetime

class TripStatus(Enum):
    REQUEST_PENDING  = "REQUEST_PENDING"
    DRIVER_ASSIGNED  = "DRIVER_ASSIGNED"
    DRIVER_EN_ROUTE  = "DRIVER_EN_ROUTE"
    DRIVER_ARRIVED   = "DRIVER_ARRIVED"
    TRIP_IN_PROGRESS = "TRIP_IN_PROGRESS"
    TRIP_COMPLETED   = "TRIP_COMPLETED"
    CANCELLED        = "CANCELLED"

# Valid state transitions
VALID_TRANSITIONS = {
    TripStatus.REQUEST_PENDING:  [TripStatus.DRIVER_ASSIGNED, TripStatus.CANCELLED],
    TripStatus.DRIVER_ASSIGNED:  [TripStatus.DRIVER_EN_ROUTE, TripStatus.CANCELLED],
    TripStatus.DRIVER_EN_ROUTE:  [TripStatus.DRIVER_ARRIVED, TripStatus.CANCELLED],
    TripStatus.DRIVER_ARRIVED:   [TripStatus.TRIP_IN_PROGRESS, TripStatus.CANCELLED],
    TripStatus.TRIP_IN_PROGRESS: [TripStatus.TRIP_COMPLETED],
    TripStatus.TRIP_COMPLETED:   [],  # Terminal state
    TripStatus.CANCELLED:        [],  # Terminal state
}

class TripStateMachine:
    
    def transition(self, trip_id, current_status, new_status, actor, metadata=None):
        # Transition valid မဟုတ်လျှင် error
        if new_status not in VALID_TRANSITIONS[current_status]:
            raise InvalidTransitionError(
                f"Cannot transition from {current_status} to {new_status}"
            )
        
        # Database update (optimistic locking)
        updated = self.db.update(
            "UPDATE trips SET status = %s, updated_at = %s "
            "WHERE trip_id = %s AND status = %s",
            (new_status.value, datetime.now(), trip_id, current_status.value)
        )
        
        if updated == 0:
            raise ConcurrentUpdateError("Trip was modified by another process")
        
        # Event emit (downstream services ကို notify)
        self.event_bus.publish(f"trip.{new_status.value.lower()}", {
            "trip_id": trip_id,
            "previous_status": current_status.value,
            "new_status": new_status.value,
            "actor": actor,
            "timestamp": datetime.now().isoformat(),
            "metadata": metadata
        })
        
        return True
```

---

## ၄၄.၇ Payment Flow (Post-Trip Charge)

```
Trip Completed Event
        |
        v
+------------------+
| Payment Service  |
+------------------+
        |
        | 1. Get saved payment method
        v
+------------------+
| Payment Method   |
| Store (Vault)    |
+------------------+
        |
        | 2. Charge via gateway
        v
+------------------+      +------------------+
| Payment Gateway  | ---> | Stripe / Braintree|
| Adapter          |      +------------------+
+------------------+
        |
        | 3. Update trip record
        v
+------------------+
| Trip Service     |
| (mark paid)      |
+------------------+
        |
        | 4. Notify rider
        v
+------------------+
| Notification     |
| Service          |
+------------------+
```

```python
class PaymentService:
    
    async def process_post_trip_payment(self, trip_id):
        trip = await self.trip_repo.get(trip_id)
        
        # Idempotency check - ထပ်မံ charge မဖြစ်စေရ
        if trip.payment_status == "CHARGED":
            return {"status": "already_charged"}
        
        # Rider ၏ payment method ရယူ
        payment_method = await self.payment_method_repo.get_default(trip.rider_id)
        
        try:
            # Payment gateway ဖြင့် charge
            charge_result = await self.gateway.charge(
                amount=trip.final_fare,
                currency="USD",
                payment_method_id=payment_method.token,
                idempotency_key=f"trip_{trip_id}_charge",  # Duplicate prevention
                description=f"Ride on {trip.created_at}"
            )
            
            # Trip record update
            await self.trip_repo.update(trip_id, {
                "payment_status": "CHARGED",
                "payment_transaction_id": charge_result.transaction_id,
                "charged_at": datetime.now()
            })
            
            # Driver earnings record ကို update
            driver_earnings = trip.final_fare * 0.75  # 75% driver ရသည်
            await self.wallet_service.credit(trip.driver_id, driver_earnings)
            
            return {"status": "success", "transaction_id": charge_result.transaction_id}
            
        except PaymentDeclinedError as e:
            # Retry logic သို့မဟုတ် alternative payment method
            await self.handle_payment_failure(trip_id, e)
```

---

## ၄၄.၈ Driver Payout — Weekly Settlement (Wallet Service)

```python
class WalletService:
    
    async def credit_driver_earnings(self, driver_id, amount, trip_id):
        """
        Trip တိုင်းပြီးတိုင်း driver earnings wallet ထဲ ထည့်သည်
        """
        # Idempotent - same trip_id ကို ထပ်မံ credit မလုပ်ရ
        existing = await self.db.get(
            "SELECT id FROM driver_earnings WHERE trip_id = %s", (trip_id,)
        )
        if existing:
            return  # Already credited
        
        await self.db.execute(
            """INSERT INTO driver_earnings 
               (driver_id, amount, trip_id, created_at)
               VALUES (%s, %s, %s, NOW())""",
            (driver_id, amount, trip_id)
        )
    
    async def process_weekly_payout(self, driver_id):
        """
        Weekly settlement - Cron job မှ trigger လုပ်
        """
        # Unpaid earnings aggregate
        total_earnings = await self.db.get_scalar(
            """SELECT SUM(amount) FROM driver_earnings 
               WHERE driver_id = %s AND payout_id IS NULL""",
            (driver_id,)
        )
        
        if total_earnings < 1.0:  # Minimum payout threshold
            return
        
        # Bank transfer via Stripe Connect
        payout = await self.stripe.create_transfer(
            amount=int(total_earnings * 100),  # cents
            currency="usd",
            destination=driver.stripe_account_id
        )
        
        # Mark earnings as paid
        await self.db.execute(
            """UPDATE driver_earnings SET payout_id = %s 
               WHERE driver_id = %s AND payout_id IS NULL""",
            (payout.id, driver_id)
        )
```

---

## ၄၄.၉ Full Architecture Diagram & Data Flow

```
+----------+  HTTPS/WS  +-------------+  Route  +------------------+
|  Rider   | ---------> | API Gateway | ------> |   User Service   |
|  App     |            +-------------+         +------------------+
+----------+                  |
                              | Route
+----------+  HTTPS/WS        v
|  Driver  | ---------> +------------------+   +------------------+
|  App     |            | Location Service | -> | Redis Geo Index  |
+----------+            +------------------+   +------------------+
                                |
                          Kafka Events
                                |
              +-----------------+------------------+
              |                 |                  |
              v                 v                  v
    +------------------+ +----------+   +------------------+
    | Matching Service | | Pricing  |   | Analytics Service|
    +------------------+ | Service  |   +------------------+
              |          +----------+
              | Match found
              v
    +------------------+
    |   Trip Service   |
    | (State Machine)  |
    +------------------+
              |
         Trip Events (Kafka)
              |
    +---------+---------+
    |                   |
    v                   v
+----------+    +------------------+
| Payment  |    | Notification     |
| Service  |    | Service          |
+----------+    +------------------+
    |
    v
+----------+
| Wallet   |
| Service  |
+----------+
```

### Key Data Flows

**1. Ride Request Flow:**
```
Rider App -> API Gateway -> Trip Service (create pending trip)
                                    |
                         -> Pricing Service (get fare estimate)
                                    |
                         -> Matching Service (find driver)
                                    |
                         -> Notification Service (notify driver)
```

**2. Driver Location Update Flow:**
```
Driver App -> WebSocket Gateway -> Location Service -> Redis Geo
                                                    -> Kafka (location events)
                                                    -> Cassandra (history)
```

**3. Payment Flow:**
```
Trip Completed -> Kafka Event -> Payment Service -> Payment Gateway
                                                 -> Wallet Service
                                                 -> Notification Service
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **Real-Time Location Tracking:** WebSocket ဖြင့် persistent connection ထားပြီး Redis Geo Index ကို location storage နှင့် proximity search အတွက် အသုံးပြုသည်။ Geohash သည် distributed caching အတွက် ကောင်းမွန်သော key scheme ဖြစ်သည်။

2. **Matching Algorithm:** Pure distance-based matching မဟုတ်ဘဲ ETA, driver rating, acceptance rate တို့ပါ ထည့်သွင်း weighted scoring ဖြင့် optimal match ရှာဖွေသည်။

3. **Surge Pricing:** Kafka stream ဖြင့် real-time demand/supply signals ကို aggregate လုပ်ကာ geohash grid level တွင် multiplier တွက်ချက်သည်။ Cache TTL 30 seconds ဖြင့် responsiveness ကို balance လုပ်သည်။

4. **State Machine:** Trip lifecycle ကို explicit state machine ဖြင့် model လုပ်ခြင်းသည် invalid transitions ကို ကာကွယ်ပြီး audit trail ကို ရိုးရှင်းစေသည်။

5. **Idempotency:** Payment processing တွင် idempotency key အသုံးပြုခြင်းသည် duplicate charges ဖြစ်ခြင်းကို ကာကွယ်သည်။ Network failures များသော distributed systems တွင် မရှိမဖြစ် လိုအပ်သည်။

6. **Event-Driven Architecture:** Services များသည် Kafka events မှတဆင့် communicate ပြုသောကြောင့် tight coupling ကင်းပြီး independent scaling ဖြစ်နိုင်သည်။

7. **Data Consistency:** Location data အတွက် eventual consistency လက်ခံနိုင်သော်လည်း payment data အတွက် strong consistency (ACID transactions) မဖြစ်မနေ လိုအပ်သည်။
