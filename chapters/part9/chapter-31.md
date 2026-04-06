# အခန်း ၃၁: Banking System

## နိဒါန်း

Banking system သည် software engineering ၏ most demanding domains ထဲမှ တစ်ခုဖြစ်သည်။ Money ကို deal ပြုလုပ်သောကြောင့် data correctness, security, compliance, နှင့် auditability တို့သည် non-negotiable requirements ဖြစ်သည်။ ဤအခန်းတွင် production-grade banking system ကို ACID transactions, double-entry bookkeeping, distributed transaction sagas, real-time fraud detection, နှင့် immutable audit logs တို့ဖြင့် မည်ကဲ့သို့ architect ပြုလုပ်မည်ကို လေ့လာမည်ဖြစ်သည်။

Banking system ၏ golden rule မှာ — "Money should never be created or destroyed; only transferred." ဤ principle ကို enforce ပြုလုပ်ရန် double-entry bookkeeping, event sourcing, နှင့် saga pattern တို့ကို combine ပြုသုံးသည်။

---

## ၃၁.၁ Requirements နှင့် Scale Estimation

### Functional Requirements

- **Account Management**: Account creation, KYC verification, account types (savings, current, loan)
- **Transaction Processing**: Deposits, withdrawals, transfers, bill payments
- **Balance Inquiry**: Real-time balance ကို check ပြုနိုင်ရမည်
- **Transaction History**: Paginated transaction history
- **Fraud Detection**: Real-time suspicious transactions detect ပြုရမည်
- **Notifications**: Transaction alerts (SMS, email, push)

### Non-Functional Requirements

- **ACID**: Atomicity, Consistency, Isolation, Durability — all transactions
- **Auditability**: Every state change ကို trace ပြုနိုင်ရမည်
- **Compliance**: PCI-DSS (payment cards), GDPR (data privacy), SOX (financial reporting)
- **Availability**: 99.999% (< 5 min/year downtime)
- **Consistency**: Strong consistency for financial transactions

### Scale Estimation

```
Users: 50 million bank customers
Daily transactions: 50M customers × 3 transactions/day = 150M/day
TPS: 150M / 86,400s ≈ 1,736 TPS (peak: 10,000 TPS)

Ledger entries: Each transaction → 2 ledger entries (double-entry)
  300M entries/day × 200 bytes = 60 GB/day
  Retention: 7 years (regulatory)
  7 years × 365 × 60 GB ≈ 153 TB

Audit log: Every API call logged
  500M events/day × 500 bytes = 250 GB/day
```

---

## ၃၁.၂ Domain Decomposition

### ၃၁.၂.၁ Account Service

Customer accounts ၏ lifecycle ကို manage ပြုသည်။

**Responsibilities:**
- Account creation/closure
- Account types: Savings, Current, Fixed Deposit
- Account status management (active, frozen, closed)
- Balance read API (projection from ledger)

### ၃၁.၂.၂ Ledger Service

Financial transactions ၏ immutable record ကို maintain ပြုသည်။

**Responsibilities:**
- Double-entry bookkeeping
- Ledger entry recording
- Balance calculation
- Transaction history

### ၃၁.၂.၃ Transaction Service

Transaction orchestration ကို saga pattern ဖြင့် manage ပြုသည်။

**Responsibilities:**
- Transaction initiation
- Saga orchestration (debit → credit → notify)
- Idempotency enforcement
- Compensation transaction execution

### ၃၁.၂.၄ KYC Service

Know Your Customer compliance ကို handle ပြုသည်။

**Responsibilities:**
- Identity document verification
- AML (Anti-Money Laundering) screening
- Periodic KYC renewal
- Risk profile assessment

### ၃၁.၂.၅ Fraud Detection Service

Real-time fraud analysis ကို streaming processing ဖြင့် ဆောင်ရွက်သည်။

**Responsibilities:**
- Transaction rule engine
- ML model scoring
- Risk score calculation
- Alert generation

### ၃၁.၂.၆ Notification Service

Transaction alerts ကို multi-channel deliver ပြုသည်။

**Responsibilities:**
- SMS via gateway (Twilio, local telcos)
- Email notifications
- Push notifications (mobile banking app)
- In-app notifications

---

## ၃၁.၃ Double-Entry Ledger Design

### Double-Entry Bookkeeping Principle

```
Every financial transaction creates TWO ledger entries:
  1. Debit entry (value leaves an account)
  2. Credit entry (value enters an account)

Sum of all debits == Sum of all credits (always balanced)

Example: Transfer $100 from Alice (Acc: 1001) to Bob (Acc: 1002)
  Entry 1: DEBIT  Account:1001 amount:100 (Alice's account decreases)
  Entry 2: CREDIT Account:1002 amount:100 (Bob's account increases)
  Net change: 0 (money conserved)
```

### Ledger Database Schema

```sql
CREATE TABLE ledger_entries (
    entry_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID NOT NULL,      -- Links debit + credit entries
    account_id     UUID NOT NULL,
    entry_type     TEXT NOT NULL,      -- 'DEBIT' or 'CREDIT'
    amount         NUMERIC(20, 4),     -- 4 decimal places (MMK cents)
    currency       CHAR(3) NOT NULL,   -- 'MMK', 'USD', 'SGD'
    balance_after  NUMERIC(20, 4),     -- Running balance snapshot
    description    TEXT,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Event sourcing: Immutable. Never UPDATE or DELETE.
    CONSTRAINT positive_amount CHECK (amount > 0)
);

-- Index for balance calculation
CREATE INDEX ON ledger_entries (account_id, created_at DESC);
CREATE INDEX ON ledger_entries (transaction_id);

-- Immutable: Revoke UPDATE/DELETE privileges
REVOKE UPDATE, DELETE ON ledger_entries FROM app_user;
```

### Balance Projection

```sql
-- Current balance from ledger (accurate but slower)
SELECT 
  SUM(CASE WHEN entry_type = 'CREDIT' THEN amount ELSE -amount END) AS balance
FROM ledger_entries
WHERE account_id = '1001' AND currency = 'MMK';

-- Optimized: Use balance_after from latest entry (O(1) lookup)
SELECT balance_after FROM ledger_entries
WHERE account_id = '1001' AND currency = 'MMK'
ORDER BY created_at DESC LIMIT 1;

-- Account balance cache in Redis (write-through)
Key: balance:{account_id}:{currency}
Value: {balance, version, as_of_timestamp}
TTL: No expiry (updated on every transaction)
```

### Event Sourcing Pattern

```
State = Replay of Events

Event log:
  [AccountOpened, Deposited(1000), Transferred(-200), Deposited(500)]

Current balance = 0 + 1000 - 200 + 500 = 1300

Benefits:
  - Complete audit trail
  - Time-travel: Balance at any point in history
  - Replay for debugging
  - Derived projections (multiple read models)
```

---

## ၃၁.၄ Concurrent Transactions ကို Handle ပြုလုပ်ခြင်း

### Optimistic Locking

```sql
CREATE TABLE accounts (
    account_id UUID PRIMARY KEY,
    balance    NUMERIC(20, 4),
    version    BIGINT DEFAULT 0,   -- Optimistic lock version
    ...
);

-- Debit operation with optimistic lock:
BEGIN;
  SELECT balance, version FROM accounts WHERE account_id = $1;
  -- balance = 1000, version = 5

  -- Check: sufficient balance
  IF balance < amount THEN RAISE 'Insufficient funds'; END IF;

  -- Update with version check
  UPDATE accounts 
  SET balance = balance - $amount, version = version + 1
  WHERE account_id = $1 AND version = 5;  -- Must match!
  
  IF affected_rows = 0 THEN
    ROLLBACK; RAISE 'Concurrent modification'; 
  END IF;
  
  INSERT INTO ledger_entries (...);
COMMIT;
-- Retry on conflict (max 3 retries with exponential backoff)
```

### Idempotency Keys

```
Problem: Network timeout → Client retries → Duplicate debit

Solution: Idempotency Key

Client sends: 
  POST /transactions
  Idempotency-Key: "idem-key-uuid-abc123"
  Body: {from_account: "1001", to_account: "1002", amount: 100}

Server:
  1. Check Redis: EXISTS idem_key:abc123
  2. HIT → Return cached response (same result, no duplicate processing)
  3. MISS → Process transaction
         → Store result in Redis: SET idem_key:abc123 {result} EX 86400
         → Return result

Idempotency key TTL: 24 hours (retry window)
Key format: client_id + request_uuid + timestamp_date
```

### Preventing Double Debit

```
Defense-in-depth approach:

Layer 1: Idempotency key (application level)
Layer 2: Optimistic locking (database level)
Layer 3: Unique constraint on transaction_id in ledger
  CREATE UNIQUE INDEX ON ledger_entries(transaction_id, entry_type);
Layer 4: Distributed lock (Redis SETNX) for critical operations
Layer 5: Saga compensation (rollback if partial failure)
```

---

## ၃၁.၅ Distributed Transaction Saga: Transfer Flow

### Saga Pattern (Choreography vs Orchestration)

Banking system တွင် **Orchestration-based Saga** ကို သုံးသည် — central coordinator (Transaction Service) ကို saga ၏ steps ကို control ပြုသည်။

### Transfer Saga Flow

```
Transfer $100 from Alice (1001) to Bob (1002)

Step 1: Validate
  Transaction Service → Account Service:
    validate_transfer(from=1001, to=1002, amount=100, currency=MMK)
  ✓ Both accounts active
  ✓ Alice has sufficient balance
  ✓ Bob's account can receive (not frozen, not at limit)
  ✓ KYC status valid for both

Step 2: Debit Sender
  Transaction Service → Ledger Service:
    debit(account=1001, amount=100, transaction_id=txn-xyz)
  ✓ Debit entry created
  ✓ Alice balance: 1300 → 1200
  → State: DEBITED (persisted in Transaction table)

Step 3: Credit Receiver
  Transaction Service → Ledger Service:
    credit(account=1002, amount=100, transaction_id=txn-xyz)
  ✓ Credit entry created
  ✓ Bob balance: 500 → 600
  → State: CREDITED

Step 4: Notify
  Transaction Service → Notification Service:
    notify_debit(user=Alice, amount=100)
    notify_credit(user=Bob, amount=100)
  ✓ SMS/push sent
  → State: COMPLETED
```

### Compensating Transactions

```
Failure Scenarios:

Scenario 1: Debit succeeds, Credit fails
  Rollback: 
    compensate_debit(account=1001, amount=100, ref_txn=txn-xyz)
    → CREDIT entry of 100 back to Alice
    → transaction status = FAILED_COMPENSATED

Scenario 2: Credit succeeds, Notify fails
  No compensation (money already transferred correctly)
  Retry notification with exponential backoff
  transaction status = COMPLETED_NOTIFY_PENDING

State Machine:
  PENDING → VALIDATING → DEBITING → CREDITING → NOTIFYING → COMPLETED
                                                              ↑
  PENDING → VALIDATING → FAILED  (validation failure)
  PENDING → VALIDATING → DEBITING → DEBIT_FAILED
  PENDING → VALIDATING → DEBITING → CREDITING → CREDIT_FAILED → COMPENSATING → COMPENSATED
```

---

## ၃၁.၆ Real-Time Fraud Detection

### Kafka Streams Rules Engine

```
Transaction Event → Kafka Topic: "transaction-events"
                 → Fraud Detection Service (Kafka Streams)

Rules Engine (applied in real-time):

Rule 1: Velocity Check
  IF count(transactions, account_id, last_1_hour) > 20
  THEN risk_score += 40, flag: HIGH_VELOCITY

Rule 2: Large Transaction
  IF amount > 10_000_000 (10M MMK)
  THEN risk_score += 30, flag: LARGE_AMOUNT, require_approval

Rule 3: Geographic Anomaly  
  IF transaction.country != account.home_country
  AND last_transaction.country == account.home_country
  AND time_diff < 2_hours (impossible travel)
  THEN risk_score += 60, flag: IMPOSSIBLE_TRAVEL

Rule 4: Dormant Account Activity
  IF account.last_transaction > 180_days
  AND current_transaction.amount > 1_000_000
  THEN risk_score += 25, flag: DORMANT_ACCOUNT_ACTIVITY

Risk Score Action:
  0-30:  ALLOW (auto-approve)
  31-60: SOFT_FLAG (allow + alert team)
  61-80: STEP_UP_AUTH (require OTP confirmation)
  81+:   BLOCK (freeze transaction + alert)
```

### ML Model Scoring (gRPC)

```
Fraud Detection Service → ML Scoring Service (gRPC, sync):
  Request:
    transaction_features: {
      amount_normalized, time_of_day, merchant_category,
      device_fingerprint, ip_risk_score, velocity_1h,
      velocity_24h, avg_transaction_amount, geo_distance
    }
  
  Response: 
    {fraud_probability: 0.87, model_version: "v2.3", latency: 2ms}

gRPC chosen over REST:
  - Binary protocol (Protobuf): 5× smaller payload
  - Streaming support
  - < 5ms latency requirement for fraud check
```

---

## ၃၁.၇ Reconciliation နှင့် EOD Settlement

### End-of-Day (EOD) Settlement

```
EOD Process (runs at midnight):

1. Freeze new transactions (10 seconds window)
2. Extract all transactions for the day
3. Reconcile with external systems:
   - Central Bank clearing system
   - Card networks (Visa, Mastercard)
   - SWIFT (international transfers)

4. Generate settlement files:
   - Net position per counterparty
   - Settlement instructions
   
5. Balance verification:
   SUM(all CREDIT entries) == SUM(all DEBIT entries)
   Any discrepancy → Alert + Manual review

6. Generate regulatory reports:
   - Cash transaction reports (CTR) > $10,000
   - Suspicious activity reports (SAR)
   - Daily balance reports to Central Bank

7. Unlock for next day trading
```

### Nostro/Vostro Reconciliation

```
Our bank sends $1M to correspondent bank (USD)
  Our nostro account: DEBIT $1M
  Expected: Correspondent bank credits our account $1M

Reconciliation check:
  Our record: Sent $1M at T+0
  SWIFT confirmation: Received at T+0 + 2 hours
  Correspondent statement: Credited $1M - fees

Any mismatch:
  → Create reconciliation exception
  → Manual investigation workflow
  → Resolution: Debit/credit adjustment entry
```

---

## ၃၁.၈ Audit Log — Immutable Tamper-Evident

### Audit Log Design

```sql
CREATE TABLE audit_log (
    log_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_time    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    event_type    TEXT NOT NULL,    -- 'TRANSACTION', 'LOGIN', 'CONFIG_CHANGE'
    actor_id      UUID,             -- User or Service that performed action
    actor_type    TEXT,             -- 'USER', 'SERVICE', 'SYSTEM'
    resource_type TEXT,             -- 'ACCOUNT', 'TRANSACTION', 'USER'
    resource_id   UUID,
    action        TEXT NOT NULL,    -- 'CREATE', 'READ', 'UPDATE', 'DELETE'
    before_state  JSONB,            -- State before change
    after_state   JSONB,            -- State after change
    ip_address    INET,
    user_agent    TEXT,
    request_id    UUID,             -- Correlation ID
    hash_chain    TEXT              -- SHA-256(prev_hash + current_entry)
);

-- Hash chaining for tamper evidence:
hash_chain[n] = SHA256(hash_chain[n-1] || event_time || event_type || resource_id || actor_id)
```

### Tamper Detection

```
Integrity verification:
  SELECT log_id, hash_chain FROM audit_log ORDER BY event_time;
  
  For each entry:
    expected_hash = SHA256(prev_hash || current_fields)
    IF expected_hash != stored_hash_chain:
      ALERT: "Audit log tampered at entry {log_id}"
      
Verification runs:
  - Every hour (automated)
  - On-demand by auditors
  - Monthly compliance reports
```

---

## ၃၁.၉ Architecture Diagram နှင့် Data Flow

### Banking System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      Customer Channels                            │
│        Mobile App / Internet Banking / ATM / Branch Teller       │
└────────────┬─────────────────────────────────────────────────────┘
             │
┌────────────▼─────────────────────────────────────────────────────┐
│              API Gateway (mTLS, Rate Limit, Auth)                  │
└──────┬──────────┬──────────────┬────────────────────┬────────────┘
       │          │              │                    │
┌──────▼──┐  ┌───▼────┐  ┌──────▼──────┐  ┌──────────▼──────────┐
│Account  │  │Ledger  │  │Transaction  │  │Fraud Detection      │
│Service  │  │Service │  │Service      │  │Service              │
│         │  │(Immut.)│  │(Saga Orch.) │  │(Kafka Streams + ML) │
└──────┬──┘  └───┬────┘  └──────┬──────┘  └──────────┬──────────┘
       │          │              │                    │
┌──────▼──────────▼──────────────▼────────────────────▼──────────┐
│                     Event Bus (Kafka)                             │
│   Topics: transactions, fraud-alerts, notifications, audit        │
└────────────────────────────────────────────────────────────────┘
       │          │              │                    │
┌──────▼──┐  ┌───▼──────┐  ┌────▼────┐  ┌────────────▼──────────┐
│PostgreSQL│  │ClickHouse │  │ Redis  │  │Notification Service   │
│(ACID)   │  │(Analytics)│  │(Cache) │  │(SMS/Email/Push)        │
└─────────┘  └───────────┘  └────────┘  └──────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────┐
│                     Audit Service                                  │
│              (Immutable Log + Hash Chain Verification)            │
└─────────────────────────────────────────────────────────────────┘
```

### Transfer Transaction Data Flow

```
1. Customer initiates transfer via Mobile App
2. API Gateway: Auth + Rate limit check
3. Transaction Service: Create saga with idempotency_key
4. Fraud Detection: Real-time risk scoring (< 50ms)
   - Rules engine: velocity, amount thresholds
   - ML model: fraud probability
   - Risk score: 25 → ALLOW
5. Transaction Service Step 1: Validate accounts (Account Service)
6. Transaction Service Step 2: Debit sender (Ledger Service)
   - PostgreSQL transaction: UPDATE balance + INSERT ledger_entry
   - Update Redis balance cache
7. Transaction Service Step 3: Credit receiver (Ledger Service)
   - PostgreSQL transaction: UPDATE balance + INSERT ledger_entry
8. Transaction Service: Publish to Kafka (transaction-events)
9. Notification Service: Consume event → Send SMS to both parties
10. Audit Service: Log all steps with hash chain
11. Analytics: Update ClickHouse (dashboard, reports)
```

---

## Key Design Decisions နှင့် Trade-offs

| Decision | ရွေးချယ်ချက် | Rationale |
|---|---|---|
| Transaction model | Double-entry + Event Sourcing | Audit trail, correctness |
| Concurrency | Optimistic locking + Idempotency | Performance vs Consistency |
| Distributed txn | Saga (Orchestration) | Reliability in microservices |
| Fraud detection | Kafka Streams + gRPC ML | Real-time, low latency |
| Audit log | Immutable + Hash chain | Tamper evidence |
| Consistency | Strong (ACID) | Money correctness |

## အနှစ်ချုပ်

Banking system design ၏ essence မှာ correctness ကို performance ထက် priority ထားခြင်းဖြစ်သည်။ Double-entry ledger ဖြင့် money conservation ကို mathematically enforce ပြုလုပ်ပြီး event sourcing ဖြင့် complete audit trail ကို maintain ပြုနိုင်သည်။ Saga pattern ဖြင့် distributed transactions ကို compensation logic ဖြင့် handle ပြုလုပ်ပြီး idempotency keys ဖြင့် duplicate operations ကို prevent ပြုနိုင်သည်။ Real-time fraud detection pipeline သည် Kafka Streams + ML model ၏ combination ဖြင့် milliseconds ထဲတွင် risk assessment ပြုလုပ်ပေးသည်။ Immutable, hash-chained audit log သည် regulatory compliance နှင့် forensic investigation ကို support ပြုနိုင်သည်။
