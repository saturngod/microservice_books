# အခန်း ၃၃: Digital Wallet System (PayPal / Paytm)

## နိဒါန်း

Digital Wallet system သည် ခေတ်မီ fintech ecosystem ၏ central hub ဖြစ်လာသည်။ PayPal သည် 430 million active users ရှိပြီး Paytm သည် India ၌ 300 million users ကို ဝန်ဆောင်မှုပေးသည်။ Myanmar တွင်လည်း KBZPay, Wave Money, MPitesan ကဲ့သို့သော digital wallets များ မြင့်တက်နေသည်။ ဤအခန်းတွင် P2P transfers, top-up flows, cashback systems, spend limits, anti-fraud measures, နှင့် ledger-based balance design တို့ကို production-grade microservices perspective မှ လေ့လာမည်ဖြစ်သည်။

Digital Wallet ၏ core promise မှာ — physical cash ၏ convenience ကို digital world ၌ recreate ပြုလုပ်ခြင်းဖြစ်သည်။ ဤ simple promise ကနောက်တွင် real-time balance consistency, concurrent transfer safety, regulatory compliance (KYC/AML), နှင့် fraud prevention စသည့် complex engineering challenges များ ရှိနေသည်။

---

## ၃၃.၁ Requirements

### Functional Requirements

- **P2P Transfer**: User A မှ User B သို့ instant money transfer
- **Top-Up**: Bank account / ATM / Agent မှ wallet ထဲသို့ fund ထည့်ခြင်း
- **Cashback**: Transaction-based rewards auto-credit
- **Linked Accounts**: Bank accounts connect ပြုနိုင်ရမည်
- **Bill Payment**: Utilities, phone bills, TV bills payment
- **QR Code Payment**: QR scan ဖြင့် payment ပြုနိုင်ရမည်
- **Transaction History**: Paginated history with filter/search

### Non-Functional Requirements

- **Consistency**: Balance consistency — negative balance မဖြစ်ရ
- **Idempotency**: Duplicate transfers prevent ပြုရမည်
- **Low Latency**: P2P transfer < 1 second (perceived)
- **High Availability**: 99.99%
- **Compliance**: KYC, AML, Central Bank regulations

### Scale Estimation

```
Users: 50 million registered, 20 million DAU
Transactions:
  - P2P transfers: 20M DAU × 2 transfers/day = 40M/day
  - Bill payments: 20M DAU × 0.5/day = 10M/day
  - Total: ~50M transactions/day
  - TPS: 50M / 86,400 ≈ 578 TPS (peak: 5,000 TPS)

Balance operations:
  - Balance checks: 20M × 20 checks/day = 400M/day ≈ 4,630/second

Storage:
  - Transaction: 500 bytes × 50M/day = 25 GB/day
  - 5 years: 45 TB
```

---

## ၃၃.၂ Domain Decomposition

### ၃၃.၂.၁ Wallet Service

Wallet entity ၏ lifecycle ကို manage ပြုသည်။

**Responsibilities:**
- Wallet creation/activation
- Wallet status (active, suspended, closed)
- Wallet limits configuration
- Linked payment methods

### ၃၃.၂.၂ Balance Service

Wallet balance ၏ accurate state ကို maintain ပြုသည်။

**Responsibilities:**
- Balance read (ultra-fast)
- Balance update (atomic)
- Ledger integration
- Balance history

### ၃၃.၂.၃ Transfer Service

P2P transfers ကို saga pattern ဖြင့် orchestrate ပြုသည်။

**Responsibilities:**
- Transfer validation
- Saga orchestration (debit → credit → notify)
- Idempotency enforcement
- Transfer status tracking

### ၃၃.၂.၄ Rewards Service

Cashback နှင့် loyalty points ကို manage ပြုသည်။

**Responsibilities:**
- Cashback rule engine
- Points calculation
- Reward expiry management
- Redemption flow

### ၃၃.၂.၅ Limits Service

Regulatory နှင့် risk-based spend limits ကို enforce ပြုသည်။

**Responsibilities:**
- Daily/monthly spend limits
- Single transaction limits
- KYC tier-based limits
- Velocity rules

### ၃၃.၂.၆ KYC Service

Customer identity verification ကို manage ပြုသည်။

**Responsibilities:**
- Document verification
- Biometric verification
- KYC tier management
- AML screening

---

## ၃၃.၃ Wallet Balance Design

### Ledger-Based Balance vs Snapshot Balance

#### Approach 1: Snapshot Balance

```sql
CREATE TABLE wallets (
    wallet_id UUID PRIMARY KEY,
    user_id   UUID UNIQUE NOT NULL,
    balance   NUMERIC(20, 4) NOT NULL DEFAULT 0,
    currency  CHAR(3) NOT NULL,
    version   BIGINT DEFAULT 0,  -- Optimistic lock
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Debit operation:
BEGIN;
  SELECT balance, version FROM wallets WHERE wallet_id = $1 FOR UPDATE;
  IF balance < amount THEN RAISE 'Insufficient funds'; END IF;
  UPDATE wallets SET balance = balance - amount, version = version + 1
  WHERE wallet_id = $1 AND version = $current_version;
  INSERT INTO transactions (...);
COMMIT;

Pros: O(1) balance read
Cons: Hot row contention for popular wallets (like merchant wallets)
```

#### Approach 2: Pure Ledger (Event Sourcing)

```sql
CREATE TABLE ledger (
    entry_id       UUID PRIMARY KEY,
    wallet_id      UUID NOT NULL,
    transaction_id UUID NOT NULL,
    entry_type     TEXT CHECK (entry_type IN ('DEBIT', 'CREDIT')),
    amount         NUMERIC(20, 4) NOT NULL CHECK (amount > 0),
    running_balance NUMERIC(20, 4) NOT NULL,  -- Balance after this entry
    created_at     TIMESTAMPTZ DEFAULT NOW()
);

-- Balance = latest running_balance
SELECT running_balance FROM ledger
WHERE wallet_id = $1
ORDER BY created_at DESC LIMIT 1;

Pros: Complete audit trail, time-travel balance
Cons: Write contention on same wallet_id (serial ledger entries)
```

#### Hybrid Approach (Production Recommended)

```
Primary Balance: Snapshot in Redis (fast reads)
  Key: wallet:balance:{wallet_id}:{currency}
  Value: {balance: 12500.50, version: 1247, as_of: "2026-04-06T10:00:00Z"}
  Strategy: Write-through (update on every transaction)

Authoritative Balance: Ledger table (PostgreSQL)
  - All transactions append to ledger
  - Redis is cache; ledger is source of truth
  - Periodic reconciliation: Redis vs Ledger

Negative Balance Prevention:
  1. Redis check: Quick balance check (10μs)
  2. Optimistic lock on PostgreSQL: Definitive check
  3. Constraint: CHECK (balance >= 0) on snapshot table
  4. Three-way defense: App logic + DB constraint + Redis guard
```

---

## ၃၃.၄ P2P Transfer Saga

### Transfer State Machine

```
States:
  INITIATED → VALIDATING → DEBITING → CREDITING → NOTIFYING → COMPLETED
                                                               
  INITIATED → VALIDATING → FAILED (validation error)
  INITIATED → VALIDATING → DEBITING → DEBIT_FAILED
  INITIATED → VALIDATING → DEBITING → CREDITING → CREDIT_FAILED 
                                                → COMPENSATING → COMPENSATED
```

### Detailed Saga Flow

```
Transfer: User A sends 5000 MMK to User B

Step 1: VALIDATING
  Transfer Service:
    a. Validate sender wallet (active, not frozen)
    b. Validate recipient wallet (active, can receive)
    c. Check sender KYC tier (sufficient for amount)
    d. Check sender daily limit (Limits Service)
    e. Check idempotency key (not duplicate)
    f. Anti-fraud check (Fraud Service)
  Result: VALIDATED → proceed to DEBITING

Step 2: DEBITING  
  Ledger Service:
    a. Acquire optimistic lock on sender's wallet
    b. Check balance ≥ 5000 MMK
    c. Create DEBIT ledger entry for sender
    d. Update sender's snapshot balance: 10,000 → 5,000 MMK
    e. Update Redis balance cache
    f. Record saga step: DEBITED
  Result: DEBITED → proceed to CREDITING

Step 3: CREDITING
  Ledger Service:
    a. Create CREDIT ledger entry for recipient
    b. Update recipient's snapshot balance: 2,000 → 7,000 MMK
    c. Update Redis balance cache
    d. Record saga step: CREDITED
  Result: CREDITED → proceed to NOTIFYING

Step 4: NOTIFYING
  Notification Service:
    a. Send push to User A: "5,000 MMK sent to User B"
    b. Send push to User B: "5,000 MMK received from User A"
    c. Send SMS (if push notification fails)
  Result: NOTIFIED → COMPLETED

Compensation (Credit failure after Debit):
  Compensating Transaction:
    a. Create CREDIT entry for sender (same amount)
    b. Update sender balance: 5,000 → 10,000 MMK
    c. Mark original transaction: COMPENSATED
    d. Notify sender: "Transfer failed, funds returned"
```

---

## ၃၃.၅ Top-Up Flow

### Bank → Wallet (via Payment Gateway)

```
User tops up 50,000 MMK via KBZ Bank:

1. User: Select "Top Up" → Choose Bank → Enter amount
2. Wallet App → Payment Gateway API: Initiate payment
3. Payment Gateway: Redirect to KBZ bank payment page
4. User: Authenticate with bank, authorize debit
5. KBZ Bank: Debit user's bank account
6. KBZ Bank → Payment Gateway: Payment success callback
7. Payment Gateway → Wallet Service: Webhook (payment.completed)
   {
     "event": "payment.completed",
     "reference": "top_up_uuid_123",
     "amount": 50000,
     "currency": "MMK",
     "status": "success"
   }
8. Wallet Service (Webhook Handler):
   a. Verify webhook signature
   b. Check idempotency (not already processed)
   c. Create CREDIT ledger entry
   d. Update wallet balance: X → X + 50,000 MMK
   e. Notify user: "50,000 MMK added to wallet"
9. Mark top_up record as COMPLETED

Idempotency:
  - Payment Gateway sends webhook with unique event_id
  - Wallet Service stores processed event_ids in Redis
  - Duplicate webhook → Return 200 OK, skip processing
```

### Agent Top-Up (Cash)

```
Agent-based top-up (for unbanked users):
  1. User visits registered agent (e-money agent)
  2. User gives cash to agent
  3. Agent: Enter user wallet ID + amount in Agent App
  4. Agent App → Wallet Service: Agent top-up request
  5. Agent's wallet debited
  6. User's wallet credited
  7. Both parties get confirmation

Agent Wallet:
  - Agents have their own wallets (funded by wholesale transfer)
  - Agent balance must be sufficient for float
  - Daily settlement: Agent's cash collected vs wallet balance
```

---

## ၃၃.၆ Cashback နှင့် Rewards Engine (Event-Driven)

### Cashback Rule Engine

```
Rules stored in database (configurable by business team):

Rule 1:
  event: TRANSACTION_COMPLETED
  merchant_category: GROCERY
  min_amount: 1000
  cashback_type: PERCENTAGE
  cashback_value: 2.0  (2%)
  max_cashback: 500
  campaign_period: 2026-01-01 to 2026-06-30
  user_eligibility: ALL_USERS

Rule 2:
  event: BILL_PAYMENT_COMPLETED
  bill_type: ELECTRICITY
  cashback_type: FIXED
  cashback_value: 200  (200 MMK fixed)
  max_per_month: 1
  user_eligibility: KYC_VERIFIED_USERS

Event Processing:
  Kafka → Rewards Consumer:
    1. Consume "transaction.completed" event
    2. Load applicable rules (cached in Redis)
    3. Calculate cashback: 2% of 5,000 = 100 MMK
    4. Check max_cashback: 100 < 500 → OK
    5. Create cashback transaction
    6. CREDIT user wallet: +100 MMK
    7. Notify user: "You earned 100 MMK cashback!"
    8. Update campaign statistics
```

### Points System

```
Points earning:
  1 point = 1 MMK spent (base rate)
  Food delivery: 2x points
  Special merchants: 5x points

Points redemption:
  100 points = 1 MMK cashback
  OR: Direct discount at checkout

Points ledger:
  Separate ledger from money ledger
  Points have expiry (12 months from earning)
  
Expiry job (daily batch):
  SELECT wallet_id, points_balance 
  FROM points_ledger 
  WHERE earned_at < NOW() - INTERVAL '12 months'
    AND status = 'ACTIVE';
  → Mark as EXPIRED, debit points balance
  → Notify user: "Your 500 points expired"
```

---

## ၃၃.၇ Daily/Monthly Spend Limits နှင့် Rate Limiting

### KYC Tier-Based Limits

```
Tier 0 (No KYC - Phone number only):
  Single transaction: 50,000 MMK max
  Daily limit: 200,000 MMK
  Monthly limit: 500,000 MMK
  Receive: 200,000 MMK/day

Tier 1 (Basic KYC - NRC/Passport):
  Single transaction: 1,000,000 MMK
  Daily limit: 5,000,000 MMK
  Monthly limit: 20,000,000 MMK

Tier 2 (Full KYC - NRC + Address proof + Income proof):
  Single transaction: 50,000,000 MMK
  Daily limit: No limit (subject to AML monitoring)
  Monthly limit: No limit

Limit Enforcement:
  Limits Service checks:
    1. Single transaction amount vs tier limit
    2. Daily spent (from Redis counter, reset at midnight)
    3. Monthly spent (from Redis counter, reset monthly)
    
  Redis counters:
    Key: limits:daily:{wallet_id}:{date}
    Value: total spent today
    TTL: 2 days (1 day after current day)
    INCRBY: atomic increment on each transaction
```

---

## ၃၃.₈ Anti-Fraud Checks

### Real-Time Fraud Signals

```
At transaction time, Fraud Service checks:

Signal 1: Account Age
  IF wallet created < 7 days AND amount > 500,000 MMK
  THEN risk_score += 30

Signal 2: Transaction Velocity
  IF count(transfers, sender_wallet, last_1_hour) > 10
  THEN risk_score += 40, flag: HIGH_VELOCITY

Signal 3: Recipient Risk
  IF recipient_wallet.created_at < 3_days
  AND this is sender's first transfer to recipient
  THEN risk_score += 25, flag: NEW_RECIPIENT

Signal 4: Behavioral Anomaly
  IF transfer_amount > avg_transfer × 5  (user's own history)
  THEN risk_score += 20, flag: UNUSUAL_AMOUNT

Signal 5: Device/IP Risk
  IF device_fingerprint not in user's known devices
  AND high-value transaction
  THEN risk_score += 30, require_step_up_auth

Score Actions:
  0-30:  AUTO_APPROVE
  31-60: SOFT_FLAG (allow + log)
  61-80: STEP_UP_AUTH (require PIN/biometric confirmation)
  81+:   BLOCK + alert fraud team

P2P Transfer Risk Pattern (Money Mule Detection):
  Rapid Receive → Rapid Forward = Suspicious
  IF wallet received > 5 transfers in 1 hour
  AND sent > 5 transfers in same hour
  THEN flag: POTENTIAL_MONEY_MULE
  → Manual review queue
```

---

## ၃၃.₉ Architecture Diagram နှင့် Data Flow

### Digital Wallet System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                       Client Layer                                │
│              Mobile App (iOS/Android) / Web App                  │
└───────────────────────┬──────────────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────────────┐
│              API Gateway (Auth + Rate Limit + TLS)                │
└──────┬───────────┬──────────────┬──────────────────┬─────────────┘
       │           │              │                  │
┌──────▼──┐  ┌────▼──────┐  ┌────▼──────┐  ┌───────▼──────────┐
│ Wallet  │  │ Transfer  │  │  Top-Up   │  │  Rewards         │
│ Service │  │ Service   │  │  Service  │  │  Service         │
│         │  │ (Saga)    │  │           │  │  (Event-Driven)  │
└──────┬──┘  └────┬──────┘  └────┬──────┘  └───────┬──────────┘
       │           │              │                  │
┌──────▼───────────▼──────────────▼──────────────────▼──────────┐
│                    Balance / Ledger Service                      │
│           (Double-Entry + Snapshot Cache)                        │
└───────────────────────┬──────────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────────┐
│                       Data Layer                               │
│  ┌───────────────┐  ┌─────────────┐  ┌──────────────────┐   │
│  │  PostgreSQL   │  │    Redis    │  │    Kafka         │   │
│  │  (Ledger +    │  │  (Balance   │  │  (Events)        │   │
│  │   Wallets)    │  │   Cache +   │  │                  │   │
│  │               │  │   Limits)   │  │                  │   │
│  └───────────────┘  └─────────────┘  └──────────────────┘   │
└───────────────────────────────────────────────────────────────┘
       │                                      │
┌──────▼──────────────┐           ┌───────────▼────────────────┐
│   Fraud Detection   │           │   Notification Service     │
│   Service           │           │   (SMS + Push + Email)     │
└─────────────────────┘           └────────────────────────────┘
       │
┌──────▼──────────────┐
│   KYC / Limits      │
│   Service           │
└─────────────────────┘
```

### P2P Transfer Data Flow

```
1. User A opens app → Enter User B's phone/wallet ID + amount
2. App → Transfer Service: POST /transfers {from, to, amount, idempotency_key}
3. API Gateway: JWT validation + rate limit
4. Transfer Service: Check idempotency key → New transfer
5. Limits Service: Check A's daily limit → OK
6. Fraud Service: Risk score = 25 → AUTO_APPROVE
7. KYC Service: Both users KYC valid
8. Balance Service: Check A's balance (Redis) → Sufficient
9. Ledger Service: BEGIN TRANSACTION
   - Create DEBIT entry for A
   - Update A's snapshot balance (atomic)
   - Commit
10. Ledger Service: BEGIN TRANSACTION
    - Create CREDIT entry for B
    - Update B's snapshot balance (atomic)
    - Commit
11. Transfer Service: Mark transfer COMPLETED
12. Kafka: Publish "transfer.completed" event
13. Notification Service: Send push to A + B
14. Rewards Service (async): Check cashback rules → Apply if eligible
15. Response to User A: "Transfer successful" (< 1 second total)
```

---

## Key Design Decisions နှင့် Trade-offs

| Decision | ရွေးချယ်ချက် | Rationale |
|---|---|---|
| Balance storage | Ledger + Redis cache | Audit trail + Performance |
| Negative balance | Multi-layer defense | Financial correctness |
| Transfer | Saga + Idempotency | Reliability |
| Cashback | Event-driven (async) | Decouple from transfer latency |
| Limits | Redis counters | Low latency enforcement |
| Fraud | Real-time rules engine | Instant decision |

## အနှစ်ချုပ်

Digital Wallet system ၏ engineering core မှာ **balance consistency** ဖြစ်သည်။ Hybrid ledger-plus-snapshot approach ဖြင့် audit trail ကို maintain ပြုရင်း O(1) balance reads ကို achieve ပြုနိုင်သည်။ P2P Transfer saga ဖြင့် distributed consistency ကို idempotency keys ဖြင့် at-most-once semantics ကို guarantee ပြုနိုင်သည်။ Cashback engine ကို event-driven architecture ဖြင့် transfer latency မထိခိုက်ဘဲ complex reward calculations ကို async process ပြုနိုင်သည်။ KYC tier-based limits နှင့် real-time fraud detection ကို combine ပြုလုပ်ခြင်းဖြင့် regulatory compliance ကို maintain ပြုပြီး user experience ကို smooth ဖြစ်စေနိုင်သည်။
