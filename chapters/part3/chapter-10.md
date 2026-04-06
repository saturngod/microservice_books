# အခန်း ၁၀ — Event Sourcing

## နိဒါန်း

ရိုးရာ data persistence နည်းလမ်းတွင် current state ကိုသာ database တွင် သိမ်းဆည်းသည်။ ဆိုလိုသည်မှာ row တစ်ခုကို update လုပ်သောအခါ မူလ value ပျောက်ကွယ်သွားသည်။ **Event Sourcing** သည် ဤ paradigm ကို လုံးဝပြောင်းလဲသည်။ Current state ကို မသိမ်းဘဲ ၎င်း state ကို ဖြစ်ပေါ်စေသော **event sequence** (ဖြစ်ရပ်စဉ်ဆက်) ကိုသာ သိမ်းဆည်းသည်။ Current state ကို ချင်သောအခါ event များအားလုံးကို ပြန်ကစားကာ (replay) တွက်ချက်ရသည်။ ဤပုံစံသည် audit trail, temporal queries, debugging နှင့် CQRS ကဲ့သို့ advanced pattern များအတွက် အဆင်ပြေသော foundation ဖြစ်သည်။

---

## ၁၀.၁ Event Sourcing ဆိုတာ ဘာလဲ

### ရိုးရာ CRUD နှင့် Event Sourcing နှိုင်းယှဉ်ချက်

```
ရိုးရာ CRUD (Current State Only):
────────────────────────────────
Time 1: INSERT account(id=1, balance=1000)
Time 2: UPDATE account SET balance=800  WHERE id=1  (1000 ဒေတာ ပျောက်!)
Time 3: UPDATE account SET balance=1500 WHERE id=1  (800 ဒေတာ ပျောက်!)

Database: [id=1, balance=1500]  ← ပြောင်းလဲမှု history မရှိ


Event Sourcing (Full History):
──────────────────────────────
Time 1: AccountOpened    { accountId:1, initialBalance:1000 }
Time 2: MoneyWithdrawn   { accountId:1, amount:200 }
Time 3: MoneyDeposited   { accountId:1, amount:700 }

Event Store: [3 events]
Current State = replay(events) = 1000 - 200 + 700 = 1500
```

### ဘာကြောင့် Event Sourcing ကို ရွေးချယ်သနည်း

- **Audit Trail** — ဘယ်သူက ဘာကို ဘယ်အချိန်မှာ ပြောင်းသည်ဆိုသည့် မှတ်တမ်းအပြည့်ရှိသည်
- **Temporal Queries** — မည်သည့်အချိန်တွင် state မည်သို့ ရှိနေသည်ကို မေးမြန်းနိုင်သည်
- **Event Replay** — bug fix ပြီးနောက် event များ ပြန်ကစား၍ state ပြင်ဆင်နိုင်သည်
- **CQRS Integration** — read model projections ကို event stream မှ build နိုင်သည်

---

## ၁၀.၂ Event Store Design

**Event Store** သည် events များကို append-only log ပုံစံဖြင့် သိမ်းဆည်းသော database ဖြစ်သည်။ Event တစ်ခုကို write ပြီးနောက် ပြင်ဆင်ခြင်း မဖြစ်သင့်ပါ (immutable)။

### Event Store Schema

```sql
-- Event Store table ဖန်တီးသည်
CREATE TABLE event_store (
    id              BIGSERIAL PRIMARY KEY,       -- sequential ID
    aggregate_id    UUID NOT NULL,               -- entity (aggregate) ID
    aggregate_type  VARCHAR(100) NOT NULL,       -- aggregate အမျိုးအစား
    event_type      VARCHAR(100) NOT NULL,       -- event အမျိုးအစား
    sequence_number INTEGER NOT NULL,            -- aggregate ၏ version
    event_data      JSONB NOT NULL,              -- event payload (JSON)
    metadata        JSONB,                       -- correlation ID စသည်
    occurred_at     TIMESTAMPTZ DEFAULT NOW(),   -- ဖြစ်ပေါ်သည့်အချိန်

    -- တူညီသော aggregate ၏ sequence_number unique ဖြစ်ရမည်
    UNIQUE (aggregate_id, sequence_number)
);

-- query performance အတွက် index
CREATE INDEX idx_aggregate ON event_store(aggregate_id, sequence_number);
```

### Event Store Repository

```python
# Event Store Repository ဥပမာ (Python)
class EventStore:

    def append_events(self, aggregate_id, events, expected_version):
        """
        Optimistic locking ဖြင့် event များ သွင်းသည်
        expected_version — concurrent write conflict ကာကွယ်ရန်
        """
        with self.db.transaction():
            # current version စစ်ဆေးသည် (concurrent write conflict)
            current_version = self._get_version(aggregate_id)
            if current_version != expected_version:
                raise ConcurrencyConflictError(
                    f"Expected version {expected_version}, "
                    f"got {current_version}"
                )

            # event တစ်ခုချင်းစီ သိမ်းဆည်းသည်
            for i, event in enumerate(events):
                self.db.execute("""
                    INSERT INTO event_store
                    (aggregate_id, aggregate_type, event_type,
                     sequence_number, event_data, metadata)
                    VALUES (%s, %s, %s, %s, %s, %s)
                """, (
                    aggregate_id,
                    event.aggregate_type,      # aggregate အမျိုးအစား
                    event.event_type,          # event အမျိုးအစား
                    expected_version + i + 1,  # sequence number
                    json.dumps(event.data),    # event payload
                    json.dumps(event.metadata) # metadata
                ))

    def load_events(self, aggregate_id, from_version=0):
        """
        aggregate ၏ event များအားလုံး ဆွဲယူသည်
        from_version — specific version မှ ဆွဲယူနိုင်သည် (snapshot အတွက်)
        """
        rows = self.db.query("""
            SELECT event_type, event_data, sequence_number
            FROM event_store
            WHERE aggregate_id = %s
              AND sequence_number > %s
            ORDER BY sequence_number ASC   -- sequence order ဖြင့် ဖတ်မည်
        """, (aggregate_id, from_version))

        return [Event.from_row(row) for row in rows]
```

---

## ၁၀.၃ Rebuilding State from Events (Replay)

Event Store မှ events တင်ပြီး current state ကို တည်ဆောက်သည်ကို **replay** ဟုခေါ်သည်။

```python
# BankAccount aggregate ဥပမာ
class BankAccount:

    def __init__(self):
        self.account_id = None
        self.balance = 0             # လက်ကျန်ငွေ
        self.is_frozen = False       # account ပိတ်ထားသလား
        self.version = 0             # current version

    @classmethod
    def load_from_events(cls, events):
        """Event sequence မှ state ပြန်တည်ဆောက်သည်"""
        account = cls()
        for event in events:
            account._apply(event)    # event တစ်ခုချင်းစီ apply
        return account

    def _apply(self, event):
        """Event တစ်ခုကို state ပြောင်းလဲရန် apply လုပ်သည်"""
        handlers = {
            'AccountOpened':    self._on_account_opened,
            'MoneyDeposited':   self._on_money_deposited,
            'MoneyWithdrawn':   self._on_money_withdrawn,
            'AccountFrozen':    self._on_account_frozen,
        }
        handler = handlers.get(event.event_type)
        if handler:
            handler(event.data)
        self.version = event.sequence_number   # version update

    def _on_account_opened(self, data):
        self.account_id = data['accountId']
        self.balance = data['initialBalance']  # ဖွင့်သည့် ငွေပမာဏ

    def _on_money_deposited(self, data):
        self.balance += data['amount']         # ငွေသွင်း

    def _on_money_withdrawn(self, data):
        self.balance -= data['amount']         # ငွေထုတ်

    def _on_account_frozen(self, data):
        self.is_frozen = True                  # account ဘလော့

    def withdraw(self, amount):
        """Command — ငွေထုတ်ရန်"""
        if self.is_frozen:
            raise AccountFrozenError("Account is frozen")  # ပိတ်ထားသော account
        if self.balance < amount:
            raise InsufficientFundsError("Not enough balance")  # ငွေမလောက်

        # event generate လုပ်သည် (state ကိုယ်တိုင် မပြောင်းဘဲ)
        event = MoneyWithdrawnEvent(
            accountId=self.account_id,
            amount=amount
        )
        self._apply_and_record(event)

    def _apply_and_record(self, event):
        self._apply(event)                     # local state update
        self.uncommitted_events.append(event)  # save ရန် queue
```

### Replay Flow

```
Event Store
┌──────────────────────────────────────────────────────┐
│ seq=1: AccountOpened    { balance: 1000 }             │
│ seq=2: MoneyWithdrawn   { amount:  200 }             │
│ seq=3: MoneyDeposited   { amount:  500 }             │
│ seq=4: MoneyWithdrawn   { amount:  100 }             │
└──────────────────────────────────────────────────────┘
          ↓ load & replay
Apply seq=1 → balance = 1000
Apply seq=2 → balance = 800
Apply seq=3 → balance = 1300
Apply seq=4 → balance = 1200

Current State: { balance: 1200, version: 4 }
```

---

## ၁၀.၄ Snapshots for Performance

Event အများကြီးရှိသောအခါ replay သည် နှေးနိုင်သည်။ **Snapshot** သည် certain version တွင် state ကို ကူးသိမ်းကာ ၎င်းနောက်မှ event များသာ replay ဖြစ်နိုင်သည်။

```python
# Snapshot table
CREATE TABLE snapshots (
    aggregate_id    UUID PRIMARY KEY,
    version         INTEGER NOT NULL,      -- snapshot ကောက်သည့် version
    state_data      JSONB NOT NULL,        -- serialized state
    taken_at        TIMESTAMPTZ DEFAULT NOW()
);

# Snapshot ပါဝင်သော load logic
class AccountRepository:

    def load(self, account_id):
        # ၁။ snapshot ရှိလျှင် ဆောင်ရွက်သည်
        snapshot = self.snapshot_store.get(account_id)

        if snapshot:
            account = BankAccount.from_snapshot(snapshot)
            # snapshot version ထက်နောက်က event များသာ load
            events = self.event_store.load_events(
                account_id,
                from_version=snapshot.version  # snapshot version မှ ဆက်ဖတ်
            )
        else:
            account = BankAccount()
            events = self.event_store.load_events(account_id)  # အစကနေ

        # ကျန်ရှိသော events replay
        for event in events:
            account._apply(event)

        return account

    def save_snapshot(self, account):
        """ကြာကြာ replay မဖြစ်ရန် periodic snapshot ကောက်သည်"""
        if account.version % 100 == 0:  # ၁၀၀ events ရောက်တိုင်း snapshot
            self.snapshot_store.save(Snapshot(
                aggregate_id=account.account_id,
                version=account.version,
                state_data=account.to_dict()    # state serialize
            ))
```

```
Without Snapshot:  replay 10,000 events → နှေး
With Snapshot:     load snapshot(v9900) + replay 100 events → မြန်
```

---

## ၁၀.၅ Temporal Queries

Event Sourcing ၏ အကြီးမားဆုံး အားသာချက်တစ်ခုသည် **temporal queries** ဖြစ်သည်၊ ဆိုလိုသည်မှာ "မည်သည့်အချိန်တွင် state မည်သို့ ရှိနေသည်" ကို မေးနိုင်သည်။

```python
# Temporal Query — ကောင်းသော use case
class AccountRepository:

    def load_at_time(self, account_id, point_in_time):
        """
        သတ်မှတ်သောအချိန်တွင် account ၏ state ကို ပြသသည်
        audit, dispute resolution, debugging တွင် အသုံးဝင်သည်
        """
        # ထို point_in_time မတိုင်မှီ events သာ load
        events = self.db.query("""
            SELECT * FROM event_store
            WHERE aggregate_id = %s
              AND occurred_at <= %s       -- သတ်မှတ်ချိန်မတိုင်မှီ
            ORDER BY sequence_number ASC
        """, (account_id, point_in_time))

        account = BankAccount()
        for event in events:
            account._apply(event)

        return account   # ထိုအချိန်က state ကို return

# သုံးပုံ ဥပမာ
# မနေ့ ညနေ ၆ နာရီတွင် account ဘယ်လောက်ရှိသလဲ?
account_yesterday = repo.load_at_time(
    "acc-001",
    datetime(2026, 4, 5, 18, 0, 0)   # တိကျသောအချိန်
)
print(f"Balance was: {account_yesterday.balance}")
```

---

## ၁၀.၆ Event Sourcing + CQRS Together

Event Sourcing သည် CQRS (Command Query Responsibility Segregation) နှင့် အတူ အမြဲလိုလို တွဲသုံးသည်။

```
Write Side (Event Sourcing):
──────────────────────────────────────────────
Command ──→ Load Aggregate (replay events)
        ──→ Validate & Execute Business Logic
        ──→ Generate Domain Events
        ──→ Save Events to Event Store

Read Side (Projections):
──────────────────────────────────────────────
Event Store ──→ Event Stream
           ──→ Projection Handler
           ──→ Read Model (denormalized, fast queries)
           ──→ Client Query ──→ Read Model
```

```python
# Projection — event မှ read model တည်ဆောက်သည်
class AccountSummaryProjection:

    def handle(self, event):
        """Event တစ်ခုချင်းစီ ရောက်လာသောအခါ read model update"""

        if event.event_type == 'AccountOpened':
            # read model ထဲ insert
            self.read_db.insert('account_summaries', {
                'account_id': event.data['accountId'],
                'balance':    event.data['initialBalance'],
                'status':     'active',
                'opened_at':  event.occurred_at
            })

        elif event.event_type == 'MoneyDeposited':
            # balance update
            self.read_db.execute("""
                UPDATE account_summaries
                SET balance = balance + %s,
                    last_transaction_at = %s
                WHERE account_id = %s
            """, (event.data['amount'],
                  event.occurred_at,
                  event.data['accountId']))

        elif event.event_type == 'MoneyWithdrawn':
            self.read_db.execute("""
                UPDATE account_summaries
                SET balance = balance - %s,
                    last_transaction_at = %s
                WHERE account_id = %s
            """, (event.data['amount'],
                  event.occurred_at,
                  event.data['accountId']))
```

### Full Architecture Diagram

```
┌─────────┐  Command   ┌───────────────┐  Events  ┌─────────────┐
│ Client  │──────────→ │ Command       │─────────→│ Event Store │
│         │            │ Handler       │          │ (append only│
│         │            └───────────────┘          └──────┬──────┘
│         │                                              │
│         │                                    Event Stream
│         │                                              │
│         │  Query     ┌───────────────┐  ←─────  ┌──────▼──────┐
│         │──────────→ │ Read Model    │          │ Projection  │
│         │            │ (fast reads)  │          │ Handler     │
└─────────┘            └───────────────┘          └─────────────┘
```

---

## အဓိကသင်ခန်းစာများ

- **Event Sourcing** သည် current state ကို မသိမ်းဘဲ ဖြစ်ပေါ်ချိန် event sequence ကိုသာ သိမ်းသည်၊ current state ကို replay ဖြင့် ရယူသည်။
- Event Store သည် **append-only** ဖြစ်သည်၊ event တစ်ခုကို write ပြီးနောက် ပြင်ဆင်ခြင်း မဖြစ်နိုင်ပါ။
- **Optimistic Locking** (sequence number) ဖြင့် concurrent write conflict ကို ကာကွယ်သည်။
- **Snapshot** သည် event count များသောအခါ replay performance ကို တိုးတက်စေသည်၊ n events တိုင်း snapshot တစ်ကြိမ် ကောက်ပါ။
- **Temporal Queries** သည် Event Sourcing ၏ လုပ်ဆောင်နိုင်မှုတစ်ခုဖြစ်ပြီး ၎င်းသည် audit, compliance နှင့် debugging တွင် အလွန်အသုံးဝင်သည်။
- Event Sourcing ကို CQRS နှင့် တွဲသုံးသောအခါ write side ကို event-sourced aggregate ဖြင့် ကိုင်တွယ်ပြီး read side ကို optimized projection ဖြင့် ကိုင်တွယ်နိုင်သည်။
