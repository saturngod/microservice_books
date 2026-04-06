# အခန်း ၃၇: Stock Trading System ဒီဇိုင်း

## နိဒါန်း

Stock trading system သည် ultra-low latency, order integrity နှင့် regulatory compliance တို့ကို တစ်ပြိုင်နက် ဖြည့်ဆည်းရသော အခက်ခဲဆုံး distributed system များထဲမှ တစ်ခုဖြစ်သည်။ Microsecond-level latency တွင် order matching ပြုလုပ်ခြင်း၊ real-time market data ကို subscriber သန်းပေါင်းများစွာသို့ fan-out ပြုလုပ်ခြင်း၊ immutable audit trail ထားရှိခြင်းတို့သည် ဤ system ၏ core challenges ဖြစ်သည်။ ဤအခန်းတွင် order book design, matching engine architecture, market data feed နှင့် settlement flow တို့ကို အသေးစိတ် ဆွေးနွေးမည်။

---

## ၃၇.၁ လိုအပ်ချက်များနှင့် Requirements အနှစ်ချုပ်

### Functional Requirements

Stock Trading Platform တစ်ခု ဆောက်လုပ်ရာတွင် အောက်ပါ အဓိက feature များ လိုအပ်သည်-

- **Order Submission:** Limit Order နှင့် Market Order ထည့်သွင်းနိုင်ရမည်
- **Order Matching:** Price-Time Priority ဖြင့် buy/sell orders တွဲဖက်ပေးနိုင်ရမည်
- **Real-Time Market Data:** Stock price, volume, OHLCV (candle) data ကို real-time ဖြင့် ပြသနိုင်ရမည်
- **Portfolio Management:** User ၏ holdings, P&L ကို ကြည့်ရှုနိုင်ရမည်
- **Trade Settlement:** Trade ပြီးနောက် T+2 settlement ပြုလုပ်နိုင်ရမည်
- **Audit Trail:** Regulatory requirement အတွက် immutable event log ထိန်းသိမ်းနိုင်ရမည်
- **Risk Management:** Pre-trade risk checks ပြုလုပ်နိုင်ရမည်

### Non-Functional Requirements

| သတ်မှတ်ချက် | ပမာဏ |
|---|---|
| Order Processing Latency | ၁ms အောက် (ultra-low latency) |
| Market Data Latency | ၁ms အောက် |
| System Uptime | ၉၉.၉၉% (Trading hours တွင်) |
| Order Throughput | ၁ မီလီယံ orders/second |
| Regulatory Compliance | SEC, MiFID II, FINRA |
| Audit Log Retention | ၇ နှစ် |

### Scale Estimation

```
Orders per day: 10 Billion (global market)
Peak trading hours: 10am - 12pm, 2pm - 4pm (4 hours peak)
Peak QPS: 10B × (4/24 ratio) ÷ 14,400 = ~116,000 orders/second

Market Data subscribers: 50 Million concurrent
Tick data per second: 1 Million price updates
Storage per day: 1M ticks × 200 bytes × 86,400 = ~17 TB/day
```

---

## ၃၇.၂ Domain Decomposition: Service များ ခွဲခြားခြင်း

### ၁. Order Management Service (OMS)
- Order validation, creation, modification, cancellation
- User account balance check
- Database: PostgreSQL (ACID transaction)

### ၂. Matching Engine Service
- Order book management
- Buy/Sell order matching
- **Single-threaded, in-memory processing** (determinism အတွက်)

### ၃. Market Data Service
- Real-time price feed distribution
- OHLCV candle aggregation
- Historical data storage

### ၄. Risk Engine Service
- Pre-trade risk checks (synchronous, sub-millisecond)
- Position limits, concentration limits
- Circuit breaker management

### ၅. Portfolio Service
- User holdings tracking
- P&L calculation
- Real-time position update

### ၆. Settlement Service
- T+2 settlement processing
- Cash and securities transfer
- Custodian bank integration

### ၇. Notification Service
- Order fill notifications
- Price alerts
- Account statements

---

## ၃၇.၃ Order Book Design

### Price-Time Priority

Stock exchange order book သည် **Price-Time Priority** (FIFO within same price level) ဖြင့် လည်ပတ်သည်-

```
BID (Buy) Side:          ASK (Sell) Side:
Price  │ Qty │ Orders   Price  │ Qty │ Orders
──────────────────       ──────────────────
$150.2 │ 500 │ [O1,O2]  $150.5 │ 300 │ [O5]
$150.1 │ 300 │ [O3]     $150.6 │ 600 │ [O6,O7]
$150.0 │ 800 │ [O4]     $150.7 │ 200 │ [O8]

Spread = $150.5 - $150.2 = $0.3
```

### Order Types

**Limit Order:** သတ်မှတ်ထားသော ဈေးနှုန်း (သို့ ပိုကောင်းသော ဈေးနှုန်း) တွင်သာ execute ပြုလုပ်သည်

**Market Order:** ရရှိနိုင်သော အကောင်းဆုံး ဈေးနှုန်းဖြင့် အမြန် execute ပြုလုပ်သည်

**Stop Order:** ဈေးနှုန်း သတ်မှတ်ချက်ရောက်မှ trigger ဖြစ်သည်

### In-Memory Order Book Data Structure

Matching Engine တွင် **Red-Black Tree** (Java TreeMap) သို့မဟုတ် **Skip List** ကို အသုံးပြုသည်-

```
Order Book Structure:
┌─────────────────────────────────────────┐
│  Red-Black Tree (Price Level Index)      │
│                                          │
│  Key: Price (Double)                     │
│  Value: PriceLevel {                     │
│    price: 150.20,                        │
│    total_qty: 500,                       │
│    orders: LinkedList<Order>  ←── FIFO   │
│  }                                       │
└─────────────────────────────────────────┘

Time Complexity:
  - Add Order: O(log N)
  - Cancel Order: O(log N) + O(1) with HashMap index
  - Best Bid/Ask lookup: O(1) - tree min/max
  - Match Order: O(log N)
```

**LinkedList** ကို Price Level အတွင်း order queue အဖြစ် အသုံးပြုပြီး **HashMap** ဖြင့် order_id → order object mapping ထိန်းသိမ်းသည်-

```python
class OrderBook:
    def __init__(self):
        self.bids = SortedDict()  # price → PriceLevel (desc)
        self.asks = SortedDict()  # price → PriceLevel (asc)
        self.order_map = {}       # order_id → Order

    def add_order(self, order: Order):
        price_level = self.bids.get(order.price, PriceLevel())
        price_level.orders.append(order)
        self.order_map[order.id] = order
```

---

## ၃၇.၄ Matching Engine Architecture

### Single-Threaded for Determinism

Matching Engine ကို **single-threaded** ဖြင့် design ပြုလုပ်သည်။ ၎င်း၏ ရည်ရွယ်ချက်-

- **Determinism:** ကြိုဆိုမှုအစဉ် (order of execution) သည် reproducible ဖြစ်ရမည်
- **No Lock Contention:** Multi-threading ဖြင့် lock ပြဿနာများ ဖြစ်နိုင်သည်
- **Audit Compliance:** "ဘယ် order ကို ဘယ်ချိန် process ပြုလုပ်သည်" ကို ကြည်လင်ဆင်ချောကြည်ဖြင့် စစ်ဆေးနိုင်ရမည်

```
Order Queue (Kafka partition - single consumer)
        │
        ▼
  ┌─────────────────────────┐
  │   Matching Engine       │
  │   (Single Thread)       │
  │                         │
  │  1. Receive Order       │
  │  2. Validate            │
  │  3. Match / Add to Book │
  │  4. Emit Trade Events   │
  └─────────────────────────┘
        │
        ▼
  Trade Event Stream (Kafka)
```

### Event Sourcing for Order State

Order state ကို direct DB update ဖြင့် မထည့်ဘဲ **Event Sourcing** pattern ကို အသုံးပြုသည်-

```
Events:
1. OrderCreated   {order_id, user_id, symbol, qty, price, type}
2. OrderMatched   {order_id, matched_order_id, qty, price}
3. OrderFilled    {order_id, fill_qty, fill_price}
4. OrderCancelled {order_id, reason}
5. OrderRejected  {order_id, reason}
```

Current order state ကို event replay ဖြင့် reconstruct ပြုလုပ်နိုင်သည်။ ၎င်းသည် audit trail အတွက် အလွန်အရေးကြီးသည်။

---

## ၃၇.၅ Real-Time Market Data Feed

### Distribution Architecture

```
Matching Engine
      │
      ▼ (Trade Events)
   Kafka Topic: "trade-events"
      │
      ├──▶ Tick Data Service     ──▶ WebSocket broadcast (raw ticks)
      │
      ├──▶ Candle Aggregator     ──▶ SSE (1m, 5m, 1h candles)
      │    (Kafka Streams / Flink)
      │
      └──▶ Market Depth Service  ──▶ WebSocket (order book updates)
```

### Fan-Out to Millions of Subscribers

**Challenge:** ၅၀ မီလီယံ concurrent user များကို market data broadcast ပြုလုပ်ရပြီး per-tick latency ကို ၁ms အောက် ထားရမည်။

**Solution: Tiered Fan-Out Architecture**

```
Level 1: Matching Engine → Kafka (1 producer)
Level 2: Kafka → Market Data Aggregators (100 instances)
Level 3: Aggregators → Connection Servers (1,000 servers)
Level 4: Connection Servers → Client WebSockets (50,000 each)
```

**OHLCV Candle Generation (Kafka Streams):**
```
Kafka Streams Job:
  Input: trade-events stream
  Tumbling Window: 1 minute
  Output: {
    symbol: "AAPL",
    open: 150.2,
    high: 151.5,
    low: 149.8,
    close: 151.0,
    volume: 2500000,
    timestamp: "2024-01-15T10:00:00Z"
  }
```

---

## ၃၇.၆ Pre-Trade Risk Checks

Trade ထည့်သွင်းခြင်းမတိုင်မီ **synchronous sub-millisecond risk checks** ပြုလုပ်သည်-

```
Order ──▶ Risk Engine ──▶ OMS ──▶ Matching Engine
           (< 1ms)

Risk Checks:
1. Sufficient balance/margin check
2. Position limit check (max shares per symbol)
3. Order size limit (whale protection)
4. Duplicate order check (fat-finger prevention)
5. Circuit breaker check (market-wide halt conditions)
6. Wash trade detection (self-trading prevention)
```

**Risk Engine Architecture:**

- User ၏ position data ကို **in-memory cache** (Redis) ထဲ သိမ်းဆည်းသည်
- Risk check ကို **CPU-optimized** code (SIMD, lock-free data structures) ဖြင့် ၁ms အောက် complete ပြုလုပ်သည်
- Check မအောင်ပါက order ကို Reject ပြုလုပ်ပြီး user ကို notify ပြုလုပ်သည်

---

## ၃၇.၇ Trade Settlement (T+2)

Trade ပြီးမြောက်ချိန်မှ ၂ business day အတွင်း settlement ပြုလုပ်ရမည်-

```
Day 0 (Trade Date):
  - Trade matched and confirmed
  - Settlement obligation recorded

Day 1:
  - Trade validation with counterparties
  - Securities movement preparation

Day 2 (Settlement Date):
  - Securities: Delivered from seller to buyer
  - Cash: Transferred from buyer to seller
  - CCP (Central Counterparty) involvement
```

**Event-Driven Settlement Flow:**

```
Trade Event ──▶ Kafka ──▶ Settlement Service
                              │
                   ┌──────────┼──────────┐
                   ▼          ▼          ▼
              Schedule    Record      Notify
              T+2 Job     Obligation  Custodian
                   │
              (2 days later)
                   │
                   ▼
              Execute Settlement
              (DvP - Delivery vs Payment)
```

---

## ၃၇.၈ Audit Trail — Immutable Event Log

Regulatory requirement (SEC Rule 17a-4) အရ trading records ကို **immutable** ဖြင့် **7 နှစ်** သိမ်းဆည်းရမည်-

### Immutable Event Log Design

**Apache Kafka** ကို permanent event log အဖြစ် အသုံးပြုသည် (retention.ms = -1 ဆိုသည်မှာ forever):

```
Kafka Topic: "audit-log" (immutable, append-only)
  - Partition by symbol
  - Replication factor: 3
  - Min-in-sync-replicas: 2
  - Retention: Permanent
```

**Tamper Evidence:** WORM (Write Once Read Many) storage သို့ archive ပြုလုပ်သည်-

```
Daily Audit Log Export:
Kafka → S3 (WORM Bucket with Object Lock)
      → Cryptographic hash per batch
      → Hash stored in separate immutable store
```

**Query Interface:**
```
Audit Query Service:
  - By order_id: Full lifecycle trace
  - By user_id + date range: All activities
  - By symbol + time range: Market activity
  - Response time: < 5 seconds (historical, not real-time)
```

---

## ၃၇.၉ Architecture Diagram နှင့် Data Flow

### Full System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     TRADING CLIENTS                              │
│              (Web, Mobile, FIX Protocol, API)                    │
└──────────────────────┬───────────────────────────────────────────┘
                       │ HTTPS / WebSocket / FIX
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                  API GATEWAY + Load Balancer                     │
└──────┬──────────┬──────────┬──────────┬──────────┬──────────────┘
       │          │          │          │          │
       ▼          ▼          ▼          ▼          ▼
  ┌─────────┐ ┌───────┐ ┌────────┐ ┌───────┐ ┌──────────┐
  │  OMS    │ │ Risk  │ │Market  │ │Portf. │ │Notif.    │
  │(Order   │ │Engine │ │ Data   │ │Service│ │Service   │
  │Mgmt Svc)│ │       │ │Service │ │       │ │          │
  └────┬────┘ └───┬───┘ └────────┘ └───┬───┘ └──────────┘
       │          │                    │
       │ Order    │ Risk OK            │
       ▼          ▼                    │
  ┌─────────────────────────┐          │
  │    MATCHING ENGINE      │          │
  │  (Single-Threaded,      │          │
  │   In-Memory Order Book) │          │
  └────────────┬────────────┘          │
               │ Trade Events          │
               ▼                       │
         ┌─────────┐                   │
         │  Kafka  │◀──────────────────┘
         │(Streams)│
         └────┬────┘
     ┌────────┼──────────┐
     ▼        ▼          ▼
 ┌───────┐ ┌──────┐ ┌─────────┐
 │Settle │ │Audit │ │Market   │
 │Service│ │ Log  │ │Data Svc │
 └───────┘ └──────┘ └────┬────┘
                          │
                    ┌─────▼──────────┐
                    │  WebSocket     │
                    │  Broadcast     │
                    │  (50M clients) │
                    └────────────────┘
```

### Order Flow (Limit Order)

```
1. User submits Buy Limit Order for AAPL @ $150.00, qty=100

2. API Gateway → OMS
   OMS validates basic parameters

3. OMS → Risk Engine (synchronous, < 1ms)
   Risk Engine: balance check OK, position limit OK
   Risk Engine → OMS: APPROVED

4. OMS → Kafka "order-queue" → Matching Engine

5. Matching Engine:
   a. Check ASK side for orders ≤ $150.00
   b. No match found → Add to BID side at $150.00
   c. Emit: OrderCreated event

6. Later, a Sell order arrives @ $150.00:
   a. Match found! Partial or full fill
   b. Emit: TradeExecuted {buyer, seller, price, qty}

7. Trade event → Kafka → Portfolio, Settlement, Notification Services

8. Portfolio Service updates Alice's holdings in real-time
9. Market Data Service broadcasts new $150.00 trade tick
10. Settlement Service schedules T+2 settlement
```

---

## Key Design Decisions နှင့် Trade-offs

| Design Decision | ရွေးချယ်မှု | ကြောင်းရင်း |
|---|---|---|
| **Matching Engine Threading** | Single-threaded | Determinism, no lock contention |
| **Order State Management** | Event Sourcing | Audit trail, state reconstruction |
| **Pre-trade Risk** | Synchronous in-memory | < 1ms requirement |
| **Market Data Distribution** | Kafka + WebSocket | Decoupled, scalable fan-out |
| **Settlement** | Event-driven, T+2 | Industry standard |
| **Audit Log Storage** | WORM S3 + Kafka | Regulatory compliance |

### ပဓာန Trade-offs

- **Latency vs Durability:** Matching Engine ကို in-memory ဖြင့် run ပြုလုပ်ခြင်းသည် latency ကောင်းသော်လည်း server crash ဖြစ်ပါက state ဆုံးရှုံးနိုင်သည်။ ၎င်းကို Kafka event log မှ state recovery ဖြင့် compensate ပြုလုပ်သည်
- **Throughput vs Consistency:** Single-threaded matching engine သည် throughput limit ရှိသည်။ Multiple matching engines ကို symbol ranges ဖြင့် ပိုင်းခြားနိုင်သည် (horizontal partitioning)
- **Simplicity vs Flexibility:** FIX Protocol ကို support ပြုလုပ်ခြင်းသည် institutional clients အတွက် လိုအပ်သော်လည်း implementation complexity တိုးသည်
