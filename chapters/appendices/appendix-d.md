# နောက်ဆက်တွဲ D: CAP Theorem နှင့် PACELC ကိုးကားလမ်းညွှန်

---

## D.1 CAP Theorem ရှင်းလင်းချက်

CAP Theorem (Brewer's Theorem) ကို Eric Brewer မှ 2000 ခုနှစ်တွင် တင်ပြခဲ့ပြီး Seth Gilbert နှင့် Nancy Lynch တို့မှ 2002 တွင် သက်သေပြခဲ့သည်။

> **Theorem:** Distributed system တစ်ခုသည် အောက်ပါ သုံးဆု၏ နှစ်ဆုသာ တစ်ချိန်တည်း အာမခံနိုင်သည်။

### သုံးဆု

| ဆု | အင်္ဂလိပ် | အဓိပ္ပာယ် |
|----|---------|---------|
| **C** | Consistency | Node တိုင်းသည် data ဖတ်ချိန်တွင် နောက်ဆုံး write ထားသော data (သို့မဟုတ် error) ကိုသာ ရသည် |
| **A** | Availability | Request တိုင်းသည် (guarantee မဖြစ်ဘဲ) non-error response ရသည်; data latest ဖြစ်ရမည် မဟုတ် |
| **P** | Partition Tolerance | Network partition (node ၂ ခု ဆက်သွယ်မရ) ဖြစ်သည့်တိုင် system သည် ဆက်လက် run သည် |

### Distributed System တွင် P ကို အမြဲ ရွေးရသည်

```
Network partition မဖြစ်ဘဲ မနေနိုင်ပါ — ကွန်ရက်ချွတ်ယွင်းမှု (packet loss,
node failure, datacenter split) သည် production တွင် ဖြစ်တတ်သည်။
ထို့ကြောင့် P တည်ဆောက်ထားသော system တွင် C နှင့် A ၏ trade-off ကိုသာ ရွေးရသည်။
```

---

## D.2 CAP ကို Diagram ဖြင့် မြင်ကွင်း

```
           Consistency (C)
                 △
                / \
               /   \
        CP   /  ??  \   CA
            /  C+A+P \
           /  possible\
          / (no network\
         /   partition) \
        ◁────────────────▷
  Partition             Availability
  Tolerance (P)              (A)
        AP
```

```
ဖြစ်နိုင်သော combination များ:

  CA  ─ Single-node database (network partition မဖြစ်ဟု assume)
  CP  ─ Consistency ကို ဦးစားပေး; partition တွင် availability ကျ
  AP  ─ Availability ကို ဦးစားပေး; partition တွင် consistency ကျ
```

---

## D.3 CP Systems — ဥပမာများနှင့် ရှင်းလင်းချက်

**CP System:** Network partition ဖြစ်ချိန်တွင် request ကို reject (သို့မဟုတ် timeout) လုပ်၍ consistency ကို ထိန်းသည်။

| System | CP ဖြစ်သောကြောင့် |
|--------|----------------|
| **HBase** | ZooKeeper ဖြင့် single master coordination; split-brain မဖြစ်ရ |
| **MongoDB** (strong read) | Primary node မှသာ read; primary မရှိလျှင် write ပိတ် |
| **Redis** (with Sentinel) | Master failover ကာလတွင် write reject |
| **Zookeeper** | Quorum-based; majority မရှိလျှင် write ငြင်းပယ် |
| **CockroachDB** | Raft consensus; partition ဖြစ်လျှင် minority partition write ငြင်းပယ် |
| **etcd** | Raft consensus; distributed config/leader election |

**CP trade-off:** Financial systems, metadata stores, configuration services တွင် သင့်လျော်သည် — data correctness ကို uptime ထက် ဦးစားပေးရသည်။

---

## D.4 AP Systems — ဥပမာများနှင့် ရှင်းလင်းချက်

**AP System:** Network partition ဖြစ်ချိန်တွင် stale data ဖြင့်မဆိုဖြေဆိုကာ availability ကို ထိန်းသည်။ Partition ပြေလျော့ပြီးနောက် data ကို sync ပြန်လုပ်သည်။

| System | AP ဖြစ်သောကြောင့် |
|--------|----------------|
| **Cassandra** | Masterless; any node သို့ write/read; tunable consistency |
| **DynamoDB** (eventual) | Global tables; eventual consistency by default |
| **CouchDB** | Multi-master replication; conflict resolution |
| **Riak** | Eventually consistent; last-write-wins |
| **Elasticsearch** | Replica lag ဖြစ်နိုင်; stale reads possible |
| **DNS** | TTL ကုန်မချင်း stale record ဖြန့်ဝေ |

**AP trade-off:** Social media feeds, shopping carts, DNS, user preferences တွင် သင့်လျော်သည် — အနည်းငယ် stale data ဖတ်မိသည်ကို ခံနိုင်သည်။

---

## D.5 PACELC Model — CAP ၏ တိုးချဲ့ Model

Daniel Abadi မှ 2012 ခုနှစ်တွင် **PACELC** ကို CAP ၏ limitation ကို ဖြည့်ဆည်းရန် တင်ပြခဲ့သည်။

> **CAP ၏ ကန့်သတ်ချက်:** Partition မဖြစ်သည့် normal operation ကာလတွင် C နှင့် A ၏ trade-off ကို CAP က မပြောဆို။

### PACELC ဖော်မြူလာ

```
If Partition (P):
    tradeoff between Availability (A) and Consistency (C)
Else (E = else, during normal operation):
    tradeoff between Latency (L) and Consistency (C)
```

### PACELC ဇယား

| Database | Partition behavior | Normal operation | PACELC Classification |
|----------|-------------------|-----------------|----------------------|
| **Cassandra** | Availability | Latency | PA/EL |
| **DynamoDB** | Availability | Latency | PA/EL |
| **CouchDB** | Availability | Latency | PA/EL |
| **MongoDB** | Consistency | Consistency | PC/EC |
| **HBase** | Consistency | Consistency | PC/EC |
| **ZooKeeper** | Consistency | Consistency | PC/EC |
| **MySQL Cluster** | Consistency | Consistency | PC/EC |
| **PNUTS** (Yahoo) | Availability | Latency | PA/EL |

---

## D.6 Partition Tolerance — လက်တွေ့တွင် ဘာဖြစ်သလဲ

### Network Partition ဆိုသည်မှာ

```
Datacenter A                    Datacenter B
┌──────────────┐                ┌──────────────┐
│  Node 1      │                │  Node 3      │
│  Node 2      │ ══× (ပြတ်သွား) │  Node 4      │
└──────────────┘                └──────────────┘
  Partition side 1                Partition side 2
```

### CP System ၏ response (MongoDB Primary ဥပမာ)

```
Network partition ဖြစ်သည် →
  Side 1: Primary ရှိ → write accept, data consistent
  Side 2: Secondary only → write reject (unavailable)
                        → stale read ငြင်းပယ် (သို့မဟုတ် error)
Partition ပြေ →
  Side 2 secondary သည် primary နှင့် sync
```

### AP System ၏ response (Cassandra ဥပမာ)

```
Network partition ဖြစ်သည် →
  Side 1: Node 1, 2 → write accept, respond with LOCAL_ONE
  Side 2: Node 3, 4 → write accept, respond with LOCAL_ONE
  (data diverged — conflict)
Partition ပြေ →
  Read repair / Anti-entropy process ဖြင့် reconcile
  Last-write-wins (LWW) သုံး conflict resolve
```

---

## D.7 Consistency Model များ

### Strong Consistency (Linear Consistency)
```
Write W → ─────────── → Read R
                         └→ R sees W always
```
- Write ပြီးနောက် read တိုင်းသည် နောက်ဆုံး value ကို ရသည်
- ဥပမာ: ZooKeeper, etcd, Spanner, CockroachDB

### Sequential Consistency
```
Process 1: W(x=1) ───────────────────────
Process 2:          W(x=2) ──────────────
Process 3 reads:    x=1 or x=2 (but all processes see same order)
```
- Global ordering ကို ထိန်းသည်; real-time ကို မထိန်းဘဲ
- ဥပမာ: Some distributed databases with vector clocks

### Causal Consistency
```
Write W1 (cause) → Write W2 (effect) →
  Reader: W2 ကိုမြင်လျှင် W1 ကိုလည်း မြင်ရမည်
  Concurrent writes: any order OK
```
- Causally related operations ၏ order ကိုသာ ထိန်းသည်
- ဥပမာ: MongoDB causal sessions, COPS (causally consistent stores)

### Eventual Consistency
```
Write W → propagate → (some time later) all nodes see W
          ↓
     Interim reads may return stale data
     Final state: all nodes converge
```
- Partition မရှိဘဲ လုံလောက်သောအချိန်ကြာပြီးနောက် nodes အားလုံး converge
- ဥပမာ: Cassandra (tunable), DynamoDB (default), DNS

### Read-Your-Writes Consistency
```
User writes X → User reads X → gets X (not stale)
Other users  → may still see old value (eventual)
```
- User သည် မိမိ write ကို ချက်ချင်း မြင်ရသည်
- ဥပမာ: Sticky session routing, MongoDB read preference `primary`

---

## D.8 Consistency Level Comparison — Cassandra ဥပမာ

| Consistency Level | Write Quorum | Read Quorum | Trade-off |
|------------------|-------------|------------|-----------|
| `ONE` | 1 node | 1 node | Fastest, lowest consistency |
| `QUORUM` | majority | majority | Balanced (strong in practice) |
| `LOCAL_QUORUM` | majority in local DC | majority in local DC | DC-local strong consistency |
| `ALL` | all nodes | all nodes | Strongest, slowest, least available |
| `ANY` | any 1 (hint OK) | — | Write always succeeds; weakest |

> **Formula:** Write + Read > Total Replicas → Strong Consistency အာမခံ
>
> **ဥပမာ:** 3 replicas; QUORUM write (2) + QUORUM read (2) = 4 > 3 → Strong ✓

---

## D.9 နိဂုံး — Trade-off Matrix

| Requirement | ရွေးသင့်သောနည်းလမ်း |
|------------|-------------------|
| Financial transactions (no stale data tolerable) | CP + Strong Consistency |
| Global low-latency reads (some staleness OK) | AP + Eventual Consistency |
| Config / metadata / leader election | CP (ZooKeeper / etcd) |
| Social feed, shopping cart, DNS | AP + Eventual Consistency |
| Audit log, event sourcing | CP + append-only log |
| Multi-region active-active | AP + CRDT / LWW conflict resolution |
