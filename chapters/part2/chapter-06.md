# အခန်း ၆ — GraphQL

## နိဒါန်း

GraphQL သည် Facebook (ယခု Meta) မှ ၂၀၁၅ ခုနှစ်တွင် open source ထုတ်ပြန်ခဲ့သော API query language နှင့် runtime တစ်ခုဖြစ်သည်။ Facebook သည် mobile apps ၏ over-fetching (လိုအပ်သည်ထက် ပိုသောဒေတာ ဆွဲ) နှင့် under-fetching (လိုအပ်သောဒေတာ ယူရန် requests များစွာ ပြုလုပ်ရ) ပြဿနာများကို ဖြေရှင်းရန် GraphQL ကို တည်ဆောက်ခဲ့သည်။ REST API တွင် endpoint တစ်ခုသည် fixed data structure တစ်ခုကို return ပြုသောကြောင့် client ၏ exact needs နှင့် ကိုက်ညီမှုမရှိနိုင်ပါ။ GraphQL တွင် client သည် ကိုယ်လိုချင်သောဒေတာကိုသာ query ပြုနိုင်သောကြောင့် bandwidth ကို ထိရောက်စွာ အသုံးပြုနိုင်သည်။ ဤအခန်းတွင် GraphQL ၏ core concepts, schema design, N+1 problem ဖြေရှင်းမှု, Federation architecture, Backend for Frontend (BFF) pattern, နှင့် REST/gRPC နှင့် ကွဲပြားချက်တို့ကို ဆွေးနွေးသွားမည်ဖြစ်သည်။

---

## ၆.၁ Queries, Mutations, Subscriptions

GraphQL ၌ operations သုံးမျိုး ရှိသည်။

### Queries (ဒေတာဖတ်ခြင်း)

```graphql
# GraphQL Query — client wants only specific fields
query GetOrder {
    order(id: "ord_123") {
        orderId
        status
        total {
            amount
            currency
        }
        # Customer details တစ်ချက်တည်း ဆွဲနိုင်သည် (separate request မလို)
        customer {
            name
            email
        }
        # Items လည်း တစ်ချက်တည်း
        items {
            productId
            quantity
            product {
                name
                imageUrl
            }
        }
    }
}
```

REST တွင် ဤ data ရရန် `/orders/123`, `/customers/456`, `/products/789` ဆိုသော endpoints သုံးခုသို့ requests သုံးကြိမ် ပြုရသော်လည်း GraphQL တွင် single request တစ်ကြိမ်သာ ပြုရသည်။

### Mutations (ဒေတာပြောင်းလဲခြင်း)

```graphql
# Mutation — Order ဖန်တီးသည်
mutation CreateOrder($input: CreateOrderInput!) {
    createOrder(input: $input) {
        orderId
        status
        total {
            amount
            currency
        }
        createdAt
    }
}

# Variables (separate JSON)
{
    "input": {
        "customerId": "cust_456",
        "items": [
            { "productId": "prod_789", "quantity": 2 }
        ]
    }
}
```

### Subscriptions (Real-Time)

```graphql
# Subscription — Order status ပြောင်းတိုင်း notification ရသည်
subscription OnOrderStatusChange($orderId: ID!) {
    orderStatusChanged(orderId: $orderId) {
        orderId
        previousStatus
        newStatus
        updatedAt
    }
}
```

```python
# Python Subscription Server (Strawberry GraphQL)
import strawberry
from typing import AsyncGenerator

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def order_status_changed(
        self,
        order_id: strawberry.ID
    ) -> AsyncGenerator["OrderStatusUpdate", None]:
        """
        WebSocket ဖြင့် real-time status updates stream ပြုသည်
        """
        async for update in order_event_bus.subscribe(order_id):
            yield OrderStatusUpdate(
                order_id=order_id,
                previous_status=update.from_status,
                new_status=update.to_status,
                updated_at=update.timestamp
            )
```

---

## ၆.၂ Schema Design နှင့် Type System

GraphQL schema သည် SDL (Schema Definition Language) ဖြင့် ရေးသားသည်။

```graphql
# schema.graphql — Complete Schema Example

scalar DateTime
scalar Decimal

# Enum types
enum OrderStatus {
    PENDING
    CONFIRMED
    SHIPPED
    DELIVERED
    CANCELLED
}

enum Currency {
    USD
    EUR
    MMK
    SGD
}

# Value types
type Money {
    amount: Decimal!    # ! = Non-nullable
    currency: Currency!
}

# Domain types
type Customer {
    id: ID!
    name: String!
    email: String!
    orders(
        first: Int = 10
        after: String
        status: OrderStatus
    ): OrderConnection!
}

type OrderItem {
    id: ID!
    product: Product!
    quantity: Int!
    unitPrice: Money!
    subtotal: Money!
}

type Order {
    id: ID!
    status: OrderStatus!
    customer: Customer!
    items: [OrderItem!]!
    total: Money!
    createdAt: DateTime!
    updatedAt: DateTime!
}

# Pagination (Relay spec)
type OrderEdge {
    cursor: String!
    node: Order!
}

type PageInfo {
    hasNextPage: Boolean!
    endCursor: String
}

type OrderConnection {
    edges: [OrderEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
}

# Input types (Mutations အတွက်)
input OrderItemInput {
    productId: ID!
    quantity: Int!
}

input CreateOrderInput {
    customerId: ID!
    items: [OrderItemInput!]!
}

# Root types
type Query {
    order(id: ID!): Order
    orders(
        customerId: ID
        status: OrderStatus
        first: Int = 20
        after: String
    ): OrderConnection!
    me: Customer
}

type Mutation {
    createOrder(input: CreateOrderInput!): Order!
    cancelOrder(orderId: ID!, reason: String): Order!
    updateOrderStatus(orderId: ID!, status: OrderStatus!): Order!
}

type Subscription {
    orderStatusChanged(orderId: ID!): Order!
}
```

**Resolver Implementation (Python Strawberry)**

```python
# resolvers.py
import strawberry
from typing import Optional, List

@strawberry.type
class Query:
    @strawberry.field
    async def order(self, id: strawberry.ID) -> Optional["Order"]:
        # DataLoader ဖြင့် N+1 ကို ကာကွယ်သည် (section 6.3 တွင် ဆွေးနွေးမည်)
        return await order_loader.load(str(id))

    @strawberry.field
    async def orders(
        self,
        customer_id: Optional[strawberry.ID] = None,
        status: Optional["OrderStatus"] = None,
        first: int = 20,
        after: Optional[str] = None
    ) -> "OrderConnection":
        return await order_service.paginate(
            customer_id=str(customer_id) if customer_id else None,
            status=status,
            first=first,
            after=after
        )
```

---

## ၆.၃ N+1 Problem နှင့် DataLoader

**N+1 Problem** ဆိုသည်မှာ GraphQL ၏ common performance pitfall တစ်ခုဖြစ်ပြီး Orders 10 ခု list ဆွဲပါက Order တစ်ခုချင်းအတွက် customer data query ထပ်ဆောင်ပြုတော့ database queries 1 (orders) + 10 (customers) = 11 ကြိမ် ဖြစ်နေမည်ဆိုသောပြဿနာဖြစ်သည်။

```
N+1 Problem Visualization:
Query: orders(first: 10) { customer { name } }

Database queries generated:
1. SELECT * FROM orders LIMIT 10           → 1 query
2. SELECT * FROM customers WHERE id = ?    → 1 query (order #1's customer)
3. SELECT * FROM customers WHERE id = ?    → 1 query (order #2's customer)
4. ...
11. SELECT * FROM customers WHERE id = ?   → 1 query (order #10's customer)

Total: 11 queries (1 + N)
```

### DataLoader Solution

**DataLoader** သည် batch + cache mechanism ဖြင့် N+1 ပြဿနာကို ဖြေရှင်းသည်။

```python
# dataloader.py
from strawberry.dataloader import DataLoader
from typing import List, Optional

async def batch_load_customers(customer_ids: List[str]) -> List[Optional[dict]]:
    """
    Customer IDs list တစ်ခုကို single query ဖြင့် batch load လုပ်သည်
    """
    # Single SQL IN query — queries N ကြိမ်မဟုတ်ဘဲ 1 ကြိမ်သာ
    customers = await customer_repo.find_by_ids(customer_ids)
    customer_map = {c.id: c for c in customers}

    # ID order ကို ထိန်းသိမ်းရမည် (DataLoader requirement)
    return [customer_map.get(cid) for cid in customer_ids]

# DataLoader instance ဖန်တီးသည်
customer_loader = DataLoader(load_fn=batch_load_customers)

# DataLoader ဖြင့် N+1 ဖြေရှင်းသော Resolver
@strawberry.type
class Order:
    customer_id: str

    @strawberry.field
    async def customer(self, info: strawberry.types.Info) -> "Customer":
        # DataLoader ဖြင့် ဆွဲသည် — batch လုပ်မည်
        return await info.context["customer_loader"].load(self.customer_id)
```

**DataLoader ဖြင့် N+1 ဖြေရှင်းချက်**

```
DataLoader Batching:
Query: orders(first: 10) { customer { name } }

Execution:
1. SELECT * FROM orders LIMIT 10           → 1 query
2. DataLoader collects: [id1, id2, ..., id10]
3. SELECT * FROM customers WHERE id IN (id1, id2, ..., id10) → 1 query

Total: 2 queries only!
```

**DataLoader with Caching**

```python
# context.py
from contextlib import asynccontextmanager

class GraphQLContext:
    def __init__(self):
        # Request တစ်ကြိမ်အတွင်း DataLoader cache မေ့ရမည်
        self.customer_loader = DataLoader(
            load_fn=batch_load_customers,
            cache=True  # Same request တွင် duplicate loads ကို cache
        )
        self.product_loader = DataLoader(
            load_fn=batch_load_products,
            cache=True
        )
        self.order_loader = DataLoader(
            load_fn=batch_load_orders,
            cache=True
        )
```

---

## ၆.၄ GraphQL Federation

**GraphQL Federation** (Apollo Federation) သည် services တစ်ခုချင်းကနေ single unified GraphQL schema ကို ဆောက်တည်နိုင်သည့် distributed GraphQL architecture ဖြစ်သည်။

```
Federation Architecture:
┌────────────────────────────────────────────────────────┐
│                     API Gateway                         │
│               (Apollo Router / Supergraph)              │
└──────────┬────────────────┬────────────────┬───────────┘
           │                │                │
           ▼                ▼                ▼
┌─────────────────┐ ┌──────────────┐ ┌─────────────────┐
│  User Subgraph  │ │ Order Subgraph│ │Product Subgraph │
│                 │ │              │ │                 │
│ type User @key  │ │ type Order   │ │ type Product    │
│  (fields:     │ │ extend type  │ │ @key(fields:    │
│   "id")         │ │ User @key    │ │ "id")           │
└─────────────────┘ └──────────────┘ └─────────────────┘
```

**Federation Schema Definition**

```graphql
# user_subgraph/schema.graphql
type User @key(fields: "id") {
    id: ID!
    name: String!
    email: String!
}

type Query {
    me: User
    user(id: ID!): User
}
```

```graphql
# order_subgraph/schema.graphql

# User type ကို extend လုပ်သည် — user subgraph မပြင်ဘဲ
extend type User @key(fields: "id") {
    id: ID! @external  # External ဆိုသည်မှာ user subgraph မှ ဆင်းသည်
    orders: [Order!]!  # Order subgraph မှ field ထည့်သည်
}

type Order @key(fields: "id") {
    id: ID!
    status: OrderStatus!
    total: Money!
    user: User!
}

# Reference resolver — User entity ကို resolve ရန်
# (Federation router မှ ခေါ်သည်)
type Query {
    _entities(representations: [_Any!]!): [_Entity]!
    order(id: ID!): Order
}
```

```python
# federation_resolver.py

# Order Subgraph တွင် User reference resolver
@strawberry.federation.type(keys=["id"])
class User:
    id: strawberry.ID

    @classmethod
    def resolve_reference(cls, id: strawberry.ID) -> "User":
        # Federation router မှ User entity resolve ခိုင်းသောအခါ
        return User(id=id)

    @strawberry.field
    async def orders(self) -> List["Order"]:
        return await order_service.get_orders_by_user(str(self.id))
```

---

## ၆.၅ BFF Pattern (Backend for Frontend)

**Backend for Frontend (BFF)** pattern ဆိုသည်မှာ client type (mobile, web, TV app) တစ်ခုချင်းအတွက် specialized GraphQL API layer ကို ဆောက်တည်သည်ဟူသော pattern ဖြစ်သည်။

```
BFF Architecture:
                    ┌──────────────────┐
                    │   Mobile BFF     │
                    │  (GraphQL)       │
 Mobile App ───────▶│  - Light payload │
                    │  - Offline sync  │
                    └──────┬───────────┘
                           │
                    ┌──────────────────┐
                    │   Web BFF        │       Microservices
 Web Browser ──────▶│  (GraphQL)       │──────▶ OrderService
                    │  - Rich data     │──────▶ UserService
                    │  - SSR friendly  │──────▶ ProductService
                    └──────┬───────────┘──────▶ PaymentService
                           │
                    ┌──────────────────┐
                    │  Partner API BFF │
 Third Party ──────▶│  (REST/GraphQL)  │
                    │  - Stable schema │
                    └──────────────────┘
```

**Mobile BFF Example**

```python
# mobile_bff/schema.py
import strawberry

@strawberry.type
class MobileOrderSummary:
    """
    Mobile app အတွက် optimized — minimal data
    """
    order_id: str
    status: str
    total_formatted: str  # "$99.99" ဖြင့် pre-formatted
    item_count: int
    status_icon: str      # Emoji/icon code for mobile display

@strawberry.type
class Query:
    @strawberry.field
    async def my_orders_summary(
        self,
        info: strawberry.types.Info
    ) -> List[MobileOrderSummary]:
        user_id = info.context["user_id"]

        # Multiple services မှ data aggregate လုပ်ပြီး
        # Mobile app ၏ exact format ဖြင့် return ပြုသည်
        orders = await order_service.get_user_orders(user_id)
        return [
            MobileOrderSummary(
                order_id=o.id,
                status=o.status,
                total_formatted=f"${o.total:.2f}",
                item_count=len(o.items),
                status_icon=get_status_icon(o.status)
            )
            for o in orders
        ]
```

---

## ၆.၆ GraphQL vs REST vs gRPC Decision Guide

```
Decision Framework
────────────────────────────────────────────────────────────────────────
Criteria              │ GraphQL  │ REST     │ gRPC
────────────────────────────────────────────────────────────────────────
Client flexibility    │ HIGH     │ LOW      │ LOW
Over/under-fetching   │ Solved   │ Problem  │ Partial
Schema/type safety    │ Strong   │ Optional │ Very Strong
Real-time (streaming) │ Subs     │ SSE/WS   │ Native stream
Performance           │ Medium   │ Medium   │ Highest
Browser support       │ Native   │ Native   │ Needs proxy
Learning curve        │ High     │ Low      │ High
Caching               │ Complex  │ Easy     │ Complex
Microservice-internal │ OK       │ OK       │ Best
Mobile apps           │ Ideal    │ OK       │ OK
Partner/3rd party API │ Good     │ Best     │ Limited
Existing tooling      │ Good     │ Excellent│ Good
────────────────────────────────────────────────────────────────────────
```

**Recommendation Matrix**

```
Scenario                          → Best Choice
──────────────────────────────────────────────────────
Public REST API for partners      → REST
Mobile app with varied data needs → GraphQL
Internal high-perf services       → gRPC
Real-time dashboard               → GraphQL (Subscriptions) or gRPC
Aggregating multiple services     → GraphQL BFF
Batch processing pipelines        → gRPC or REST
Event-driven notifications        → REST + Webhooks / GraphQL Subs
Multi-team micro-frontends        → GraphQL Federation
```

**Hybrid Pattern (Real World)**

```python
# hybrid_api.py
# Real-world apps တွင် တစ်ခုတည်းသာ မသုံးပါ

# Public API  → REST
# Mobile BFF  → GraphQL
# Internal    → gRPC
# Events      → Kafka

# api_router.py
from fastapi import FastAPI

app = FastAPI()

# REST endpoints (external)
app.include_router(rest_router, prefix="/api/v1")

# GraphQL BFF (mobile/web frontend)
# Strawberry GraphQL ကို mount လုပ်သည်
app.add_route("/graphql", graphql_app)
app.add_websocket_route("/graphql/ws", graphql_app)

# gRPC Server (internal service communication)
# Separate port တွင် serve လုပ်သည်
# python -m grpc_tools.protoc သုံးပြီး generate
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **GraphQL** သည် client ကိုယ်တိုင် data shape ကို ဆုံးဖြတ်နိုင်သောကြောင့် over-fetching နှင့် under-fetching ပြဿနာ ဖြေရှင်းနိုင်သည်; multiple services မှ data တစ်ချက်တည်း aggregate လုပ်နိုင်သည်။
2. **Schema-first** design — SDL ဖြင့် schema ရေးပြီး resolvers implement လုပ်ပါ; schema သည် API contract ဖြစ်သည်။
3. **N+1 Problem** — Nested relationships query ပြုသောအခါ database queries exponential တိုးနိုင်ချေရှိသည်; **DataLoader** ဖြင့် batch loading ပြုကာ single query ဖြင့် ဖြေရှင်းပါ။
4. **GraphQL Federation** — မတူညီသော teams ကိုယ်စီ GraphQL subgraph ဆောက်ပြီး Apollo Router ဖြင့် unified supergraph ဖန်တီးနိုင်သည်; large microservices ecosystem အတွက် ကောင်းမွန်သည်။
5. **BFF Pattern** — Client type (mobile, web, partner) တစ်ခုချင်းအတွက် tailored API layer ဆောက်ပါ; "one API fits all" မဟုတ်ပါ။
6. **Decision Guide** — GraphQL ကို mobile/frontend-heavy apps, flexible queries လိုသောနေရာ; REST ကို public APIs, partner integrations; gRPC ကို internal high-performance services တွင် ကိုင်ပါ; hybrid approach ကို real-world တွင် မကြာခဏ ကိုင်တွယ်ရသည်။
