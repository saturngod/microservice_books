# အခန်း ၃၄ — E-Commerce Order System (Amazon ကဲ့သို့)

## နိဒါန်း

E-Commerce Order System သည် modern distributed systems ၏ most complex use cases ထဲမှ တစ်ခုဖြစ်သည်။ Amazon ၏ Black Friday event တွင် တစ်မိနစ်လျှင် 100,000 orders ကျော်ကို process ပြုလုပ်သည်။ ဤ scale တွင် inventory management, payment processing, fulfillment orchestration, shipping coordination, နှင့် real-time notifications တို့ကို fault-tolerant microservices architecture ဖြင့် manage ပြုလုပ်ရသည်မှာ extraordinary engineering challenge ဖြစ်သည်။

ဤအခန်းတွင် order lifecycle saga, inventory oversell prevention, flash sale architecture, product search, dynamic pricing, recommendation engine, နှင့် full architecture diagram တို့ကို comprehensive microservices design perspective မှ လေ့လာမည်ဖြစ်သည်။

---

## ၃၄.၁ Requirements

### Functional Requirements

- Product catalog browse/search ပြုလုပ်နိုင်ရမည်
- Shopping cart manage ပြုလုပ်နိုင်ရမည်
- Order place ပြုလုပ်နိုင်ရမည် (checkout flow)
- Payment process ပြုလုပ်နိုင်ရမည်
- Inventory real-time tracking ပြုလုပ်နိုင်ရမည်
- Order status tracking ပြုလုပ်နိုင်ရမည်
- Product reviews/ratings ပေးနိုင်ရမည်
- Personalized recommendations ပြသနိုင်ရမည်
- Flash sale/deals support ပြုလုပ်နိုင်ရမည်

### Non-Functional Requirements

| Metric | Target |
|---|---|
| Peak Orders | 100,000 orders/minute (Black Friday) |
| Product Catalog | 500 million products |
| Search Latency | < 200ms (p99) |
| Order Placement | < 2 seconds end-to-end |
| Inventory Accuracy | 99.99% (zero oversell) |
| System Availability | 99.99% (52 min downtime/year) |
| Daily Active Users | 300 million |

### Scale Estimation

```
Order throughput:
  100K orders/min = 1,667 orders/sec (peak)
  Normal: ~500 orders/sec
  Peak-to-normal ratio: ~3.3x

Product search:
  300M DAU * 15 searches/day = 4.5B searches/day
  4.5B / 86,400 = ~52,000 searches/sec
  Peak: 3x = ~156,000 searches/sec

Inventory writes:
  Each order: avg 3 items = 3 inventory decrements
  Peak: 1,667 * 3 = 5,000 inventory writes/sec
  Flash sale: 50,000+ writes/sec (single product)

Storage:
  500M products * 5KB avg = 2.5 TB (catalog)
  Orders: 1B orders/year * 2KB = 2 TB/year
  Reviews: 500M reviews * 500B = 250 GB
```

---

## ၃၄.₂ Domain Decomposition

```
E-Commerce Domain Decomposition:
┌─────────────────────────────────────────────────────────────────┐
│                     API Gateway (Kong)                          │
└───┬────┬────┬────┬────┬────┬────┬────┬────┬────┬───────────────┘
    │    │    │    │    │    │    │    │    │    │
┌───▼──┐ │ ┌─▼──┐ │ ┌──▼─┐ │ ┌──▼──┐│ ┌──▼──┐ │
│Prodct│ │ │Cart│ │ │Ordr│ │ │Pymnt││ │Ship │ │
│ Svc  │ │ │Svc │ │ │Svc │ │ │ Svc ││ │ Svc │ │
└──────┘ │ └────┘ │ └────┘ │ └─────┘│ └─────┘ │
    ┌────▼──┐┌────▼──┐┌────▼──┐┌────▼──┐┌─────▼──┐
    │Invent ││Pricng ││Fulfil ││Notify ││Review  │
    │ Svc   ││ Svc   ││ Svc   ││ Svc   ││ Svc    │
    └───────┘└───────┘└───────┘└───────┘└────────┘
                                    ┌──────────┐
                                    │Recommend │
                                    │  Svc     │
                                    └──────────┘
```

| Service | Responsibility | Database |
|---|---|---|
| **Product Service** | Catalog CRUD, categories | PostgreSQL + Elasticsearch |
| **Inventory Service** | Stock levels, warehouse routing | Redis + PostgreSQL |
| **Cart Service** | Shopping cart management | Redis (session-like) |
| **Order Service** | Order lifecycle, saga orchestration | PostgreSQL |
| **Pricing Service** | Dynamic pricing, coupons, tax | PostgreSQL + Redis cache |
| **Payment Service** | Payment processing, refunds | PostgreSQL (ACID) |
| **Fulfillment Service** | Warehouse pick/pack/ship | PostgreSQL |
| **Shipping Service** | Carrier integration, tracking | PostgreSQL + external APIs |
| **Notification Service** | Email, SMS, push notifications | Redis (queue) + templates |
| **Review Service** | Ratings, reviews, Q&A | MongoDB (flexible schema) |
| **Recommendation Service** | Personalized suggestions | ML model + Redis cache |

---

## ₃₄.₃ Order Lifecycle Saga

### Order Saga — Orchestration Pattern

```
Order Lifecycle Saga:
┌──────────────────────────────────────────────────────────────────┐
│                    ORDER SAGA ORCHESTRATOR                       │
│                                                                  │
│  ┌─────────┐   ┌──────────┐   ┌─────────┐   ┌──────────┐      │
│  │ Place   │──>│ Reserve  │──>│ Charge  │──>│ Fulfill  │      │
│  │ Order   │   │Inventory │   │ Payment │   │          │      │
│  └─────────┘   └──────────┘   └─────────┘   └──────────┘      │
│       │              │              │              │             │
│       │              │              │         ┌────▼─────┐      │
│       │              │              │         │  Ship    │      │
│       │              │              │         │          │      │
│       │              │              │         └────┬─────┘      │
│       │              │              │              │             │
│       │              │              │         ┌────▼─────┐      │
│       │              │              │         │ Deliver  │      │
│       │              │              │         │          │      │
│       │              │              │         └──────────┘      │
│                                                                  │
│  Compensation (Failure Path):                                    │
│  Charge fails → Release Inventory → Cancel Order → Notify User  │
│  Ship fails → Refund Payment → Release Inventory → Cancel Order │
└──────────────────────────────────────────────────────────────────┘
```

```python
# Order Saga Orchestrator
from enum import Enum
import uuid

class OrderState(Enum):
    PENDING = "PENDING"
    INVENTORY_RESERVED = "INVENTORY_RESERVED"
    PAYMENT_CHARGED = "PAYMENT_CHARGED"
    FULFILLMENT_STARTED = "FULFILLMENT_STARTED"
    SHIPPED = "SHIPPED"
    DELIVERED = "DELIVERED"
    CANCELLED = "CANCELLED"
    COMPENSATION_IN_PROGRESS = "COMPENSATION_IN_PROGRESS"

class OrderSagaOrchestrator:
    """Order lifecycle saga manage ပြုလုပ်သည်
    Forward steps + compensation (rollback) logic ပါဝင်သည်"""

    def __init__(self, inventory_svc, payment_svc, fulfillment_svc,
                 shipping_svc, notification_svc):
        self.inventory = inventory_svc
        self.payment = payment_svc
        self.fulfillment = fulfillment_svc
        self.shipping = shipping_svc
        self.notification = notification_svc

    async def execute(self, order):
        """Saga forward execution — step by step ပြုလုပ်သည်"""
        saga_log = []

        try:
            # Step 1: Inventory reserve ပြုလုပ်သည်
            reservation = await self.inventory.reserve(
                items=order.items,
                order_id=order.id
            )
            saga_log.append(("inventory_reserved", reservation.id))
            order.state = OrderState.INVENTORY_RESERVED

            # Step 2: Payment charge ပြုလုပ်သည်
            payment = await self.payment.charge(
                user_id=order.user_id,
                amount=order.total_amount,
                order_id=order.id,
                # Idempotency key — retry safe
                idempotency_key=f"order-{order.id}-payment"
            )
            saga_log.append(("payment_charged", payment.id))
            order.state = OrderState.PAYMENT_CHARGED

            # Step 3: Fulfillment start ပြုလုပ်သည်
            fulfillment = await self.fulfillment.create(
                order_id=order.id,
                items=order.items,
                warehouse_id=reservation.warehouse_id
            )
            saga_log.append(("fulfillment_started", fulfillment.id))
            order.state = OrderState.FULFILLMENT_STARTED

            # Step 4: Notification send ပြုလုပ်သည်
            await self.notification.send(
                user_id=order.user_id,
                template="order_confirmed",
                data={"order_id": order.id, "eta": fulfillment.estimated_delivery}
            )

            return order

        except Exception as e:
            # Compensation — reverse order ဖြင့် rollback ပြုလုပ်သည်
            await self._compensate(order, saga_log, str(e))
            raise

    async def _compensate(self, order, saga_log, reason):
        """Compensation logic — saga steps ကို reverse ပြုလုပ်သည်"""
        order.state = OrderState.COMPENSATION_IN_PROGRESS

        for step_name, step_id in reversed(saga_log):
            try:
                if step_name == "fulfillment_started":
                    await self.fulfillment.cancel(step_id)
                elif step_name == "payment_charged":
                    await self.payment.refund(
                        payment_id=step_id,
                        reason=f"Order cancelled: {reason}"
                    )
                elif step_name == "inventory_reserved":
                    await self.inventory.release(step_id)
            except Exception as comp_error:
                # Compensation failure — manual intervention required
                # Dead letter queue သို့ push ပြုလုပ်ပြီး ops team notify
                await self._send_to_dead_letter(
                    order.id, step_name, step_id, str(comp_error)
                )

        order.state = OrderState.CANCELLED
        # User notification — order cancelled
        await self.notification.send(
            user_id=order.user_id,
            template="order_cancelled",
            data={"order_id": order.id, "reason": reason}
        )
```

---

## ₃₄.₄ Inventory Management

### Oversell Prevention

Flash sale ကဲ့သို့ high-concurrency scenario တွင် oversell (stock ထက်ပိုရောင်းမိခြင်း) ကို prevent ပြုလုပ်ရန် Redis atomic operations အသုံးပြုသည်။

```python
# Redis atomic inventory decrement — oversell prevention
import redis

class InventoryService:
    """Redis atomic operations ဖြင့် inventory manage ပြုလုပ်သည်
    Race condition free — oversell impossible"""

    def __init__(self, redis_client):
        self.redis = redis_client

        # Lua script — atomic check-and-decrement
        # Redis server ပေါ်တွင် atomic ဖြင့် execute ဖြစ်သည်
        self.reserve_script = self.redis.register_script("""
            -- KEYS[1] = inventory key (e.g., "inventory:P001")
            -- ARGV[1] = requested quantity
            -- ARGV[2] = reservation ID
            -- ARGV[3] = TTL seconds for reservation

            local current_stock = tonumber(redis.call('GET', KEYS[1]) or '0')
            local requested = tonumber(ARGV[1])

            if current_stock >= requested then
                -- Stock sufficient — atomic decrement
                redis.call('DECRBY', KEYS[1], requested)

                -- Reservation record create (TTL ဖြင့် auto-expire)
                local reservation_key = 'reservation:' .. ARGV[2]
                redis.call('HMSET', reservation_key,
                    'product_key', KEYS[1],
                    'quantity', ARGV[1],
                    'status', 'reserved'
                )
                redis.call('EXPIRE', reservation_key, ARGV[3])

                return 1  -- Success
            else
                return 0  -- Insufficient stock
            end
        """)

    def reserve_stock(self, product_id, quantity, reservation_id, ttl=600):
        """Stock reserve ပြုလုပ်သည် — atomic operation ဖြင့်"""
        key = f"inventory:{product_id}"
        result = self.reserve_script(
            keys=[key],
            args=[quantity, reservation_id, ttl]
        )
        if result == 0:
            raise InsufficientStockError(
                f"Product {product_id}: insufficient stock for qty {quantity}"
            )
        return True

    def release_stock(self, reservation_id):
        """Reservation cancel — stock ပြန်ထည့်သည်"""
        res_key = f"reservation:{reservation_id}"
        reservation = self.redis.hgetall(res_key)
        if reservation and reservation[b"status"] == b"reserved":
            product_key = reservation[b"product_key"].decode()
            quantity = int(reservation[b"quantity"])
            # Atomic increment — stock ပြန်ထည့်
            self.redis.incrby(product_key, quantity)
            self.redis.hset(res_key, "status", "released")
```

### Warehouse Routing

```python
# Warehouse routing — closest warehouse ရွေးချယ်ခြင်း
class WarehouseRouter:
    """Order items ကို optimal warehouse မှ fulfill ပြုလုပ်ရန် route"""

    def route_order(self, order_items, customer_location):
        """Items ကို closest warehouse ဖြင့် fulfill ပြုလုပ်မည့် plan ရေးဆွဲ"""
        fulfillment_plan = []

        for item in order_items:
            # Stock ရှိသော warehouses ရှာသည်
            available_warehouses = self.inventory.get_warehouses_with_stock(
                product_id=item.product_id,
                min_quantity=item.quantity
            )

            if not available_warehouses:
                raise OutOfStockError(f"No warehouse has {item.product_id}")

            # Customer location နှင့် closest warehouse ရွေးချယ်
            best_warehouse = min(
                available_warehouses,
                key=lambda w: self._calculate_shipping_score(
                    warehouse=w,
                    customer=customer_location,
                    quantity=item.quantity
                )
            )

            fulfillment_plan.append({
                "item": item,
                "warehouse_id": best_warehouse.id,
                "estimated_days": self._estimate_delivery_days(
                    best_warehouse, customer_location
                )
            })

        return fulfillment_plan

    def _calculate_shipping_score(self, warehouse, customer, quantity):
        """Distance + cost + capacity ပေါင်းစပ်ပြီး score တွက်သည်"""
        distance = self._haversine_distance(
            warehouse.lat, warehouse.lng,
            customer.lat, customer.lng
        )
        # Shipping cost weight 40%, distance 40%, capacity utilization 20%
        return (distance * 0.4 +
                warehouse.shipping_cost_per_unit * quantity * 0.4 +
                warehouse.utilization_rate * 0.2)
```

---

## ₃₄.₅ Flash Sale Architecture

```
Flash Sale Architecture:
┌──────────┐    ┌───────────────┐    ┌──────────────┐    ┌──────────┐
│  Client  │───>│  Rate Limiter │───>│ Token Bucket │───>│  Queue   │
│(millions)│    │  (per-user)   │    │  (global)    │    │ (SQS)    │
└──────────┘    └───────────────┘    └──────────────┘    └────┬─────┘
                                                               │
                                                         ┌─────▼─────┐
                                                         │  Worker   │
                                                         │  Pool     │
                                                         │ (metered) │
                                                         └─────┬─────┘
                                                               │
                                                         ┌─────▼─────┐
                                                         │  Redis    │
                                                         │ Atomic    │
                                                         │ Inventory │
                                                         └───────────┘

1. Rate Limiter: user per second 1 request ခွင့်ပြု (abuse prevention)
2. Token Bucket: global rate 10,000 req/sec ကန့်သတ် (system protection)
3. Queue: requests queue ထဲထည့်ပြီး controlled rate ဖြင့် process
4. Worker: queue မှ consume ပြီး inventory check + order create
5. Redis Atomic: Lua script ဖြင့် oversell prevention
```

```python
# Flash sale with queue metering + cache pre-warming
class FlashSaleService:
    """Flash sale architecture — queue metering pattern"""

    def __init__(self, redis_client, sqs_client):
        self.redis = redis_client
        self.sqs = sqs_client

    async def pre_warm_cache(self, sale_id, product_id, total_stock):
        """Flash sale မတိုင်ခင် cache pre-warm ပြုလုပ်သည်
        Database hit avoid ပြုလုပ်ရန်"""
        # Redis ထဲသို့ stock load ပြုလုပ်
        self.redis.set(f"flash_sale:{sale_id}:stock", total_stock)
        self.redis.set(f"flash_sale:{sale_id}:status", "ACTIVE")

        # Product data cache warm
        product = await self.product_repo.get(product_id)
        self.redis.setex(
            f"product:{product_id}",
            3600,  # 1 hour TTL
            json.dumps(product.to_dict())
        )

    async def handle_purchase_request(self, sale_id, user_id, product_id):
        """Flash sale purchase request handle ပြုလုပ်သည်"""

        # Step 1: Duplicate check — user already purchased စစ်ဆေး
        if self.redis.sismember(f"flash_sale:{sale_id}:buyers", user_id):
            raise DuplicatePurchaseError("Already purchased in this sale")

        # Step 2: Stock check (fast path — Redis read)
        remaining = int(self.redis.get(f"flash_sale:{sale_id}:stock") or 0)
        if remaining <= 0:
            raise SoldOutError("Flash sale item sold out")

        # Step 3: Queue ထဲသို့ request ထည့် (metered processing)
        message = {
            "sale_id": sale_id,
            "user_id": user_id,
            "product_id": product_id,
            "timestamp": datetime.utcnow().isoformat(),
            "request_id": str(uuid.uuid4()),
        }

        self.sqs.send_message(
            QueueUrl=f"https://sqs.../flash-sale-{sale_id}",
            MessageBody=json.dumps(message),
            # Deduplication — same user same sale same product
            MessageDeduplicationId=f"{sale_id}:{user_id}:{product_id}",
            MessageGroupId=product_id,
        )

        return {"status": "QUEUED", "message": "Processing your order..."}

    async def process_queued_request(self, message):
        """Queue consumer — controlled rate ဖြင့် orders process"""
        sale_id = message["sale_id"]
        user_id = message["user_id"]
        product_id = message["product_id"]

        # Atomic inventory decrement (Lua script)
        try:
            self.inventory.reserve_stock(
                product_id=product_id,
                quantity=1,
                reservation_id=message["request_id"]
            )

            # Mark user as buyer (dedup)
            self.redis.sadd(f"flash_sale:{sale_id}:buyers", user_id)

            # Create order
            order = await self.order_service.create(
                user_id=user_id,
                items=[{"product_id": product_id, "quantity": 1}],
                source="flash_sale",
                sale_id=sale_id
            )

            # Notify user — order confirmed
            await self.notification.send(user_id, "flash_sale_success", {
                "order_id": order.id
            })

        except InsufficientStockError:
            # Stock exhausted — notify user
            await self.notification.send(user_id, "flash_sale_sold_out", {
                "product_id": product_id
            })
```

---

## ₃₄.₆ Product Catalog & Search

```python
# Elasticsearch product search
from elasticsearch import Elasticsearch

class ProductSearchService:
    """Elasticsearch ဖြင့် product search + filtering"""

    def __init__(self):
        self.es = Elasticsearch(["http://es-cluster:9200"])
        self.index = "products"

    def search(self, query, filters=None, page=1, size=20):
        """Product search — full-text + filters + relevance scoring"""
        body = {
            "query": {
                "bool": {
                    "must": [
                        {
                            "multi_match": {
                                "query": query,
                                # Product name, description, category ကို search
                                "fields": ["name^3", "description", "category^2"],
                                "type": "best_fields",
                                "fuzziness": "AUTO",  # Typo tolerance
                            }
                        }
                    ],
                    "filter": self._build_filters(filters or {}),
                }
            },
            # Aggregations — faceted search (price range, brand, rating)
            "aggs": {
                "price_ranges": {
                    "range": {
                        "field": "price",
                        "ranges": [
                            {"to": 25}, {"from": 25, "to": 50},
                            {"from": 50, "to": 100}, {"from": 100}
                        ]
                    }
                },
                "brands": {
                    "terms": {"field": "brand.keyword", "size": 20}
                },
                "avg_rating": {
                    "avg": {"field": "rating"}
                }
            },
            "from": (page - 1) * size,
            "size": size,
            "sort": [
                {"_score": "desc"},        # Relevance score
                {"sales_count": "desc"},   # Popular items boost
            ]
        }

        result = self.es.search(index=self.index, body=body)
        return {
            "total": result["hits"]["total"]["value"],
            "products": [hit["_source"] for hit in result["hits"]["hits"]],
            "aggregations": result["aggregations"],
        }

    def _build_filters(self, filters):
        """Search filters build — price, brand, rating, category"""
        filter_clauses = []
        if "min_price" in filters:
            filter_clauses.append({"range": {"price": {"gte": filters["min_price"]}}})
        if "max_price" in filters:
            filter_clauses.append({"range": {"price": {"lte": filters["max_price"]}}})
        if "brand" in filters:
            filter_clauses.append({"term": {"brand.keyword": filters["brand"]}})
        if "min_rating" in filters:
            filter_clauses.append({"range": {"rating": {"gte": filters["min_rating"]}}})
        if "in_stock" in filters and filters["in_stock"]:
            filter_clauses.append({"term": {"in_stock": True}})
        return filter_clauses
```

---

## ₃₄.₇ Dynamic Pricing

```python
# Dynamic pricing engine — demand, competition, inventory based
class DynamicPricingEngine:
    """Real-time dynamic pricing — multiple signals ပေါင်းစပ်"""

    def calculate_price(self, product_id, base_price):
        """Product ၏ dynamic price calculate ပြုလုပ်သည်"""
        signals = self._gather_signals(product_id)

        # Demand factor: high demand → price increase
        demand_factor = self._demand_factor(signals["view_count"], signals["purchase_rate"])

        # Inventory factor: low stock → price increase
        inventory_factor = self._inventory_factor(signals["stock_level"])

        # Competition factor: competitor price compare
        competition_factor = self._competition_factor(
            base_price, signals["competitor_prices"]
        )

        # Time factor: time-of-day, day-of-week patterns
        time_factor = self._time_factor()

        # Final price calculation — weighted average of factors
        adjustment = (
            demand_factor * 0.3 +
            inventory_factor * 0.25 +
            competition_factor * 0.3 +
            time_factor * 0.15
        )

        # Price bounds: base price ၏ -20% to +50% range
        final_price = base_price * (1 + adjustment)
        final_price = max(base_price * 0.8, min(base_price * 1.5, final_price))

        return round(final_price, 2)

    def _demand_factor(self, view_count, purchase_rate):
        """Demand signal: views + conversion rate"""
        if purchase_rate > 0.1:  # 10% conversion — high demand
            return 0.15  # 15% price increase signal
        elif purchase_rate > 0.05:
            return 0.05
        return 0.0

    def _inventory_factor(self, stock_level):
        """Low stock → higher price"""
        if stock_level < 10:
            return 0.2   # Very low stock — 20% increase
        elif stock_level < 50:
            return 0.1
        elif stock_level > 500:
            return -0.1  # Overstock — 10% decrease
        return 0.0
```

---

## ₃₄.₈ Recommendation Engine

```python
# Collaborative filtering recommendation
class RecommendationEngine:
    """User-based collaborative filtering + item-based hybrid"""

    def get_recommendations(self, user_id, limit=20):
        """User အတွက် personalized recommendations"""

        # Strategy 1: Collaborative filtering — similar users' purchases
        cf_items = self._collaborative_filtering(user_id, limit=limit)

        # Strategy 2: Item-based — recently viewed items ၏ similar items
        recent_views = self.redis.lrange(f"recent_views:{user_id}", 0, 9)
        item_based = self._item_based_similarity(recent_views, limit=limit)

        # Strategy 3: Trending — popular items in user's categories
        user_categories = self._get_user_preferred_categories(user_id)
        trending = self._get_trending_in_categories(user_categories, limit=limit)

        # Merge + deduplicate + rank
        all_recommendations = self._merge_and_rank(
            cf_items, item_based, trending,
            weights=[0.4, 0.35, 0.25]  # CF gets highest weight
        )

        return all_recommendations[:limit]

    def _collaborative_filtering(self, user_id, limit):
        """Similar users ရှာပြီး ၎င်းတို့ ဝယ်ထားသော items recommend"""
        # User-item interaction matrix
        user_purchases = self._get_user_purchase_history(user_id)

        # Similar users ရှာသည် (cosine similarity)
        similar_users = self._find_similar_users(user_id, user_purchases)

        # Similar users ဝယ်ထားပြီး current user မဝယ်ရသေးသော items
        recommendations = []
        for similar_user, similarity_score in similar_users:
            their_purchases = self._get_user_purchase_history(similar_user.id)
            new_items = their_purchases - user_purchases
            for item in new_items:
                recommendations.append({
                    "product_id": item,
                    "score": similarity_score,
                    "reason": "similar_users_bought"
                })

        return sorted(recommendations, key=lambda x: x["score"], reverse=True)[:limit]
```

---

## ₃₄.₉ Architecture Diagram

```
E-Commerce Order System — Full Architecture:
┌─────────────────────────────────────────────────────────────────────────┐
│                            CDN (CloudFront)                             │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────────┐
│                         API Gateway (Kong)                              │
│              Auth │ Rate Limit │ Routing │ SSL Termination              │
└──┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬───────────────────┘
   │    │    │    │    │    │    │    │    │    │    │
┌──▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──────────┐
│Prod ││Inv ││Cart││Ord ││Pric││Pay ││Fulf││Ship││Noti││Rev ││Recommend │
│Svc  ││Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││   Svc    │
└──┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└────┬─────┘
   │     │     │     │     │     │     │     │     │     │        │
┌──▼──┐┌─▼──┐┌─▼──┐┌─▼──┐┌─▼──┐┌─▼──┐┌─▼──┐┌─▼──┐┌─▼──┐┌─▼──┐┌───▼────┐
│PgSQL││Redis││Redis││PgSQL││PgSQL││PgSQL││PgSQL││PgSQL││Redis││Mongo││ML Model│
│+ES  ││+PgSQ││     ││     ││+Rds ││     ││     ││+APIs││Queue││DB   ││+Redis  │
└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└────────┘
                         │
                    ┌────▼────────────────────────┐
                    │     Kafka Event Bus          │
                    │  (Order Events, Inventory    │
                    │   Events, Analytics Events)  │
                    └─────────────────────────────┘

Event Flow:
  Order Created → Kafka → Inventory Reserve → Payment Charge → Fulfillment
  Payment Failed → Kafka → Inventory Release → Order Cancelled → Notification
  Order Shipped → Kafka → Tracking Update → Notification → Analytics
```

---

## Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Flash Sale** | Queue + Redis Atomic | Fairness + zero oversell |
| **Inventory** | Redis + Lua script | Atomic operations, < 1ms |
| **Search** | Elasticsearch | Rich query, faceted search |
| **Order Saga** | Orchestration pattern | Central coordinator, easier debugging |
| **Recommendations** | Collaborative filtering | Personalization quality |
| **Dynamic Pricing** | Event-driven rules engine | Real-time market response |
| **Event Bus** | Kafka | Durability, replay, high throughput |

---

## အဓိက အချက်များ (Key Takeaways)

1. **Black Friday scale** (100K orders/min) ကို handle ပြုလုပ်ရန် queue-based metering + Redis atomic operations ဖြင့် inventory accuracy maintain ပြုလုပ်သည်

2. **Order lifecycle saga** ဖြင့် distributed transaction ကို compensation logic ဖြင့် reliable ဖြစ်အောင် orchestrate ပြုလုပ်သည် — payment failure ဖြစ်ပါက inventory release + order cancel ပြုလုပ်သည်

3. **Oversell prevention** — Redis Lua script ဖြင့် atomic check-and-decrement ပြုလုပ်ပြီး race condition eliminate ပြုလုပ်သည်

4. **Flash sale** — rate limiting + token bucket + queue metering pattern ဖြင့် millions of concurrent requests ကို fair + controlled manner ဖြင့် process ပြုလုပ်သည်

5. **Product search** — Elasticsearch full-text search + faceted aggregations ဖြင့် < 200ms response time ရရှိသည်

6. **Recommendation engine** — collaborative filtering + item-based similarity + trending ကို hybrid approach ဖြင့် personalization quality maximize ပြုလုပ်သည်

---

> **နောက်အခန်း Preview:** အခန်း ၃၅ တွင် Food Delivery System (DoorDash/Uber Eats) — real-time driver dispatch, ETA prediction, surge pricing တို့ကို ဆွေးနွေးသွားမည်ဖြစ်သည်။
