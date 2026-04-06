# အခန်း ၃၂: Payment Gateway System (Stripe / Razorpay)

## နိဒါန်း

Payment Gateway system သည် merchants (ကုန်သည်များ) နှင့် financial institutions (banks, card networks) ကြားတွင် bridge အဖြစ် ဆောင်ရွက်သော critical infrastructure ဖြစ်သည်။ Stripe, Razorpay, Adyen ကဲ့သို့သော systems များသည် တစ်နေ့လျှင် billions of dollars ကို process ပြုလုပ်ကြသည်။ ဤအခန်းတွင် payment lifecycle (authorization → capture → settlement), card tokenization, 3DS authentication, idempotency, retry strategies, multi-currency support, webhook delivery, reconciliation, နှင့် chargeback management တို့ကို production-grade microservices perspective မှ လေ့လာမည်ဖြစ်သည်။

Payment Gateway ၏ core challenge မှာ — **10,000+ transactions per second** ကို **sub-200ms latency** ဖြင့် process ပြုလုပ်ရပြီး **zero data breach** (PCI-DSS compliance) ကို maintain ပြုရခြင်းဖြစ်သည်။

---

## ၃၂.၁ Requirements နှင့် Scale Estimation

### Functional Requirements

- **Payment Processing**: Credit/debit cards, bank transfers, digital wallets
- **Authorization**: Card issuer မှ approval ရယူခြင်း
- **Capture**: Authorized amount ကို collect ပြုလုပ်ခြင်း
- **Settlement**: Funds ကို merchant account သို့ transfer ပြုလုပ်ခြင်း
- **Refunds**: Full/partial refund processing
- **Chargeback**: Dispute management
- **Webhooks**: Real-time event notifications to merchants
- **Tokenization**: Cardholder data ကို securely store ပြုလုပ်ခြင်း

### Non-Functional Requirements

- **PCI-DSS Level 1**: Highest security standard for payment data
- **Throughput**: 10,000 TPS (peak: 50,000 TPS)
- **Latency**: < 200ms P99 for authorization
- **Availability**: 99.999% (< 5 min/year downtime)
- **Durability**: Zero payment data loss

### Scale Estimation

```
Transactions:
  - Normal: 10,000 TPS
  - Peak (Black Friday): 50,000 TPS
  - Daily volume: 10,000 × 86,400 = 864 million transactions/day

Payment Volume:
  - Average transaction: $50
  - Daily volume: 864M × $50 = $43.2 billion/day

Storage:
  - Transaction record: 2 KB
  - 864M × 2 KB = 1.7 TB/day
  - 7 years retention = 4.4 PB

Webhook:
  - 864M transactions × 3 events each = 2.6 billion webhooks/day
  - Peak: 150,000 webhooks/second
```

---

## ၃၂.၂ Domain Decomposition

### ၃၂.၂.၁ Payment Orchestrator

Central coordinator ဖြစ်ပြီး payment lifecycle ကို manage ပြုသည်။

**Responsibilities:**
- Payment request validation
- Routing to appropriate acquirer
- Saga orchestration
- State machine management

### ၃၂.၂.၂ Acquirer Adapter Service

Different acquiring banks/networks နှင့် integrate ပြုသည်။

**Responsibilities:**
- Acquirer API integration (Visa, Mastercard, local banks)
- Protocol translation (ISO 8583, REST, SOAP)
- Circuit breaker per acquirer
- Failover to backup acquirer

### ၃၂.၂.၃ Risk Engine

Fraud assessment ကို real-time ဆောင်ရွက်သည်။

**Responsibilities:**
- Rules-based fraud detection
- ML model scoring
- Velocity checks
- 3D Secure decision (challenge vs frictionless)

### ၃၂.၂.၄ Tokenization Service

PCI scope ကို reduce ပြုသည်။

**Responsibilities:**
- Card number → Token mapping
- Secure token vault (HSM)
- Token lifecycle management
- Network token management (Visa Token Service, Mastercard MDES)

### ၃၂.၂.၅ Settlement Service

Funds movement ကို acquirer/merchant ကြားတွင် manage ပြုသည်။

**Responsibilities:**
- Settlement batch processing
- Net settlement calculation
- Fee deduction
- Payout to merchants

### ၃၂.၂.၆ Webhook Service

Merchant notifications ကို reliable deliver ပြုသည်။

**Responsibilities:**
- Event queue management
- Delivery with retry/backoff
- Signature verification
- Delivery status tracking

---

## ၃၂.၃ Payment Lifecycle

### Authorization → Capture → Settlement → Refund → Chargeback

```
┌─────────────────────────────────────────────────────────────────┐
│                      Payment Lifecycle                            │
│                                                                   │
│  Customer                                                         │
│  purchases     ┌─AUTHORIZATION─┐  ┌─CAPTURE────┐  ┌─SETTLEMENT─┐│
│  ────────────→ │ Card verified  │→ │ Funds held │→ │ Merchant   ││
│                │ Funds reserved │  │ collected  │  │ receives   ││
│                └───────────────┘  └────────────┘  │ funds      ││
│                     |                              └────────────┘│
│                   (cancel/                              |         │
│                    expire)                           (dispute)   │
│                                                    ┌─CHARGEBACK─┐│
│                                  ┌─REFUND────┐     │ Customer   ││
│                                  │ Merchant  │     │ disputes   ││
│                                  │ returns   │     │ charge     ││
│                                  │ funds     │     └────────────┘│
│                                  └───────────┘                   │
└─────────────────────────────────────────────────────────────────┘

Authorization Hold Duration:
  - Hotel/Car rental: 7-30 days
  - E-commerce: 7 days
  - Restaurant (tip adjust): 3 days
  - Default: 7 days
```

### Authorization Request Flow

```
1. Customer enters card details at checkout
2. Merchant → Payment Gateway: AuthorizeRequest
3. Risk Engine: Fraud score (< 30ms)
4. Tokenization: Card number → Token (if new card)
5. 3DS Check: Risk-based decision
   - Low risk → Frictionless flow
   - High risk → Challenge flow (OTP/biometric)
6. Gateway → Acquirer: ISO 8583 authorization message
7. Acquirer → Card Network (Visa/Mastercard): Route to issuer
8. Issuer: Check available funds, velocity limits, fraud rules
9. Issuer → Acquirer: Auth code OR decline code
10. Acquirer → Gateway: Authorization response
11. Gateway → Merchant: Success/Failure response
12. Database: Record authorization with auth_code

Total time: < 200ms (goal)
  - Risk engine: 10ms
  - Network round-trip to issuer: 100-150ms
  - Gateway processing: 40ms
```

---

## ၃၂.၄ Card Tokenization (PCI-DSS Scope Reduction)

### PCI-DSS Challenge

PCI-DSS (Payment Card Industry Data Security Standard) သည် cardholder data (card number, CVV, expiry) ကို store ပြုသော systems အားလုံးကို strict security requirements ဖြင့် audit ပြုလုပ်သည်။

### Tokenization Solution

```
Original PAN (Primary Account Number): 4111 1111 1111 1111
                      ↓  Tokenization (HSM)
Gateway Token:        tok_live_4xKj8mN2pQr

Storage:
  - Token Vault (HSM): PAN ↔ Token mapping (highly secured)
  - Merchant/Gateway: Store only token (not PAN)
  - PCI scope: Only Token Vault is in scope

Benefits:
  - Merchant systems = PCI scope reduction
  - Token useless if stolen (cannot charge without vault)
  - Network tokenization: Visa/MC replace PAN in network

Network Token (Visa Token Service):
  - PAN: 4111 1111 1111 1111
  - Device Token: 4900 0000 0000 0001 (different per device)
  - Dynamic CVV: Changes with each transaction
  - Even better security than static tokenization
```

---

## ၃၂.၅ 3DS Authentication Flow

### 3D Secure 2.0 (EMV 3DS)

```
3DS Flow Types:

1. Frictionless Flow (Low Risk):
   Gateway → 3DS Server → Issuer ACS: Risk assessment
   Issuer: "Low risk, approve without challenge"
   → Payment proceeds without user interaction
   
2. Challenge Flow (High Risk):
   Gateway → 3DS Server → Issuer ACS: Risk assessment
   Issuer: "Challenge required"
   → User sees: Enter OTP / Biometric / PIN
   → User authenticates
   → Issuer confirms authentication
   → Payment proceeds

Data sent in 3DS (100+ data points):
  - Device fingerprint (screen size, timezone, language)
  - Transaction history with this merchant
  - Shipping/billing address match
  - Browser headers
  → Issuer uses ML model on these signals

Liability Shift:
  Without 3DS: Merchant liable for fraud
  With 3DS: Issuer liable for fraud (shifted)
  → Strong incentive for merchant to use 3DS
```

---

## ၃၂.၆ Idempotency in Payment APIs

### Why Idempotency is Critical

```
Problem:
  1. Merchant sends "charge $100" to gateway
  2. Network timeout — merchant doesn't know if succeeded
  3. Merchant retries "charge $100"
  4. DANGER: Customer charged twice!

Solution: Idempotency Keys

Merchant request:
  POST /v1/payments
  Idempotency-Key: "merchant-order-id-12345"
  {amount: 100, currency: "USD", payment_method: "tok_..."}

Gateway processing:
  1. CHECK: SELECT * FROM idempotency_keys WHERE key = "merchant-order-id-12345"
  2. HIT: Return original response (no double charge)
  3. MISS: Process payment, store result with key
     INSERT INTO idempotency_keys (key, response, created_at) 
     VALUES ("merchant-order-id-12345", $result, NOW())
     ON CONFLICT (key) DO NOTHING  -- Concurrent safety

  4. Return result (same for both original + retry)

Key design:
  - Unique per merchant + order combination
  - Merchant-generated (not gateway)
  - Expiry: 24 hours (retry window)
  - Distributed lock during processing (prevent race condition)
```

---

## ၃၂.၇ Retry Strategy for Transient Failures

### Retry with Exponential Backoff + Jitter

```
Transient failures:
  - Network timeouts (not acquirer decline)
  - 5xx errors from acquirer
  - Connection reset

Retry Strategy:
  MAX_RETRIES = 3
  BASE_DELAY = 1 second
  MAX_DELAY = 30 seconds
  JITTER_FACTOR = 0.3  (random 0-30% variation)

  Attempt 1: 0s (immediate)
  Attempt 2: 1s × 2^1 + jitter = 2-2.6s
  Attempt 3: 1s × 2^2 + jitter = 4-5.2s
  → Total max wait: ~8 seconds

Non-retriable failures (do NOT retry):
  - 400 Bad Request (invalid card data)
  - Card declined (insufficient funds, stolen card)
  - Do Not Honor (issuer hard decline)

Retry vs New Auth:
  - Same authorization: Retry with SAME idempotency key
  - New payment attempt: New idempotency key (customer re-initiates)
```

---

## ၃၂.၈ Multi-Currency နှင့် Real-Time FX Rate Service

### Currency Handling

```
Payment in JPY, Merchant in USD:

1. Customer: Pays ¥15,000 (Japanese Yen)
2. Gateway FX Service: Get USD/JPY rate = 150.25
3. Calculation: ¥15,000 / 150.25 = $99.83 USD
4. DCC (Dynamic Currency Conversion):
   - Gateway offers: "Pay in USD ($99.83) or JPY (¥15,000)?"
   - Customer choice → Stored in transaction record

FX Rate Service:
  - Source: Bloomberg/Reuters FX feed (real-time)
  - Cache in Redis with 60-second TTL
  - Fallback: Last known rate if feed unavailable
  - Rate locking: Lock rate at authorization (not settlement)
  
  Rate arbitrage protection:
  - Settlement at locked authorization rate (not settlement time rate)
  - Merchant bears FX risk between auth and settlement
  - Alternative: Real-time settlement (instant DDA)
```

---

## ၃၂.၉ Webhook Delivery — Reliable At-Least-Once with Backoff

### Webhook Architecture

```
Event Sources → Kafka → Webhook Service → Merchant Endpoints

Webhook Service Components:

1. Event Consumer (Kafka):
   - Consume payment events
   - Enqueue to delivery queue (PostgreSQL)

2. Delivery Engine:
   - Worker pool (100 workers)
   - Try deliver to merchant webhook URL
   - Record attempt + response

3. Retry Scheduler:
   - Exponential backoff with jitter
   - Max 24 hours retry window

Delivery Table:
  CREATE TABLE webhook_deliveries (
    delivery_id   UUID PRIMARY KEY,
    event_id      UUID NOT NULL,
    merchant_id   UUID NOT NULL,
    endpoint_url  TEXT NOT NULL,
    payload       JSONB NOT NULL,
    signature     TEXT,          -- HMAC-SHA256 for verification
    status        TEXT,          -- PENDING, DELIVERED, FAILED, EXPIRED
    attempt_count INTEGER DEFAULT 0,
    next_retry_at TIMESTAMPTZ,
    created_at    TIMESTAMPTZ DEFAULT NOW()
  );

Retry Schedule:
  Attempt 1: Immediate
  Attempt 2: 5 minutes
  Attempt 3: 30 minutes
  Attempt 4: 2 hours
  Attempt 5: 6 hours
  Attempt 6: 24 hours
  → Status: EXPIRED (merchant must query API for missed events)
```

### Webhook Security

```
Signature verification:
  secret = merchant_webhook_secret (stored in vault)
  payload = JSON body bytes
  signature = HMAC-SHA256(secret, timestamp + "." + payload)
  
  Header: "Stripe-Signature: t=1712345678,v1=abc123..."

Merchant verification (Python):
  expected = hmac.new(secret, f"{timestamp}.{payload}", sha256)
  if not hmac.compare_digest(received, expected):
    reject  # Possible replay or tampering
```

---

## ၃၂.၁၀ Reconciliation Service

### Multi-Level Reconciliation

```
Level 1: Internal Reconciliation (Real-time)
  Compare: Transaction table vs Ledger entries
  Expected: Every authorization → ledger debit entry
  Check: Every capture → ledger credit entry
  Alert: Any mismatch within 5 minutes

Level 2: Acquirer Reconciliation (Daily)
  Compare: Gateway records vs Acquirer settlement file
  Acquirer sends: Daily batch file (ISO 8583 or CSV)
  Match by: Authorization code, transaction ID
  Exceptions:
    - Gateway auth, no acquirer record → Void authorization
    - Acquirer record, no gateway record → Investigation
    - Amount mismatch → Settlement exception

Level 3: Scheme Reconciliation (Daily)
  Compare: Acquirer records vs Card scheme (Visa/MC) clearing files
  Purpose: Ensure fees correctly calculated
  
Reconciliation Dashboard:
  - Match rate: >99.99% target
  - Exception count by type
  - Aged exceptions (> 3 days = escalate)
```

---

## ၃၂.၁၁ Dispute နှင့် Chargeback Management

### Chargeback Flow

```
Timeline:
  T+0: Transaction completed
  T+60 days: Customer disputes with issuing bank
             (Reason: fraud, not received, not as described)
  T+60: Issuer → Acquirer → Gateway: Chargeback notification
  T+60: Gateway → Merchant: Alert + Request evidence
  T+75: Merchant submits evidence (max 15 days to respond):
         - Order confirmation, delivery proof
         - Terms of service
         - Customer communication history
  T+90: Issuer reviews evidence
  T+105: Decision:
         - Merchant wins: Chargeback reversed, funds returned
         - Merchant loses: Funds permanently taken, $25 fee added

Gateway Support:
  - Automatic pre-chargeback alerts (Visa RDR, MC Consumer Clarity)
  - Evidence collection workflow for merchants
  - Chargeback rate monitoring (> 1% → Merchant at risk of termination)
```

---

## ၃၂.၁၂ Architecture Diagram နှင့် Data Flow

### Payment Gateway Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     Merchant Applications                       │
│          Web Checkout / Mobile SDK / API Integration            │
└──────────────────────┬─────────────────────────────────────────┘
                       │ HTTPS (TLS 1.3)
┌──────────────────────▼─────────────────────────────────────────┐
│                   API Gateway (mTLS)                             │
│         Rate limiting, Authentication, DDoS protection           │
└──────┬──────────┬──────────────┬──────────────────┬────────────┘
       │          │              │                  │
┌──────▼──┐  ┌───▼──────┐  ┌────▼──────┐  ┌───────▼──────────┐
│Payment  │  │Tokenization│ │  Risk    │  │  Webhook         │
│Orchestra│  │Service    │  │  Engine  │  │  Service         │
│-tor     │  │(HSM-based)│  │(ML+Rules)│  │(Kafka+Retry)     │
└──────┬──┘  └───────────┘  └──────────┘  └──────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────┐
│                 Acquirer Adapter Service                      │
│      ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │
│      │ Visa Net │  │MC Network│  │  Local Bank (KBZ etc) │  │
│      │ Adapter  │  │ Adapter  │  │       Adapter        │  │
│      └──────────┘  └──────────┘  └──────────────────────┘  │
└────────────────────────────────────────────────────────────┘
       │
┌──────▼─────────────────────────────────────────────────────┐
│                    Data Layer                                 │
│  PostgreSQL (transactions) + Redis (idempotency cache)       │
│  + Vault (token storage) + Kafka (event streaming)           │
└──────────────────────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────┐
│              Settlement + Reconciliation Service              │
│         (Daily batch + Real-time monitoring)                  │
└────────────────────────────────────────────────────────────┘
```

### Payment Authorization Data Flow

```
1. Customer enters card → Merchant sends to gateway API
2. API Gateway: Auth + rate limit check
3. Tokenization Service: Generate/lookup token for PAN
4. Risk Engine: Calculate fraud score (10ms)
5. 3DS Service: Frictionless/Challenge decision
   - If challenge: Customer authenticates
6. Payment Orchestrator: Create payment record (status: AUTHORIZING)
7. Acquirer Adapter: Send ISO 8583 auth request to acquirer
8. Acquirer → Card Network → Issuer: Route + validate
9. Issuer Response: Approved (auth_code) / Declined
10. Acquirer Adapter: Receive response
11. Payment Orchestrator: Update payment record (AUTHORIZED/DECLINED)
12. Ledger Service: Record authorization hold entry
13. Kafka: Publish payment.authorized event
14. Webhook Service: Queue webhook for merchant
15. API Response: Return to merchant (< 200ms)
```

---

## Key Design Decisions နှင့် Trade-offs

| Decision | ရွေးချယ်ချက် | Rationale |
|---|---|---|
| PCI compliance | Tokenization + HSM | Reduce scope, security |
| Concurrency | Idempotency keys | Exactly-once semantics |
| Reliability | At-least-once webhooks | No missed events |
| Fraud | Risk Engine + 3DS | Liability shift + accuracy |
| Acquirer routing | Circuit breaker + failover | High availability |
| Currency | Rate lock at auth time | Predictable merchant revenue |
| Retry | Exponential backoff + jitter | Avoid thundering herd |

## အနှစ်ချုပ်

Payment Gateway system ၏ complexity မှာ security (PCI-DSS), reliability (idempotency), scalability (50,000 TPS), နှင့် compliance (3DS, reconciliation) ကို simultaneously maintain ပြုရခြင်းဖြစ်သည်။ Tokenization ဖြင့် PCI scope ကို dramatically reduce ပြုနိုင်ပြီး idempotency keys ဖြင့် network unreliability ကြောင့် ဖြစ်ပေါ်သော duplicate charges ကို prevent ပြုနိုင်သည်။ Webhook delivery system ၏ at-least-once guarantee ဖြင့် merchant notifications ကို reliable deliver ပြုနိုင်ပြီး reconciliation service ဖြင့် financial accuracy ကို verify ပြုနိုင်သည်။ ဤ system တည်ဆောက်ရာတွင် "trust but verify" principle ကို — authorization မှ settlement ထိ ဆောင်ရွက်ရသည်မှာ payment engineering ၏ fundamental approach ဖြစ်သည်။
