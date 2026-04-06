# အခန်း ၃၈: Live Sports နှင့် Gaming Leaderboard System ဒီဇိုင်း

## နိဒါန်း

Live sports score update နှင့် gaming leaderboard system များသည် real-time data ကို subscriber သန်းပေါင်းများစွာထံ sub-second latency ဖြင့် ပေးပို့ရသော high-throughput system များဖြစ်သည်။ Score ingestion, Redis Sorted Set-based leaderboard ranking, fan-out notification နှင့် anti-cheat detection တို့ကို ဤအခန်းတွင် microservices architecture ဖြင့် ဒီဇိုင်းရေးဆွဲပုံကို လေ့လာမည်။

---

## ၃၈.၁ လိုအပ်ချက်များနှင့် Requirements အနှစ်ချုပ်

### Functional Requirements

- **Real-Time Score Updates:** Game တွင် score ပြောင်းလဲသည်နှင့် တပြိုင်တည်း အားလုံးသို့ broadcast ပြုလုပ်ရမည်
- **Global Leaderboard:** Player rankings ကို real-time ဖြင့် ပြသနိုင်ရမည် (Top 100, Top 1000)
- **Player Ranking Lookup:** မိမိ rank ကို (ဥပမာ: #4,523,891) ကြည့်ရှုနိုင်ရမည်
- **Event Replay:** Game replay ကို later viewing အတွက် သိမ်းဆည်းနိုင်ရမည်
- **Anti-Cheat Detection:** ဆေးလိမ်ခြင်း၊ bot အသုံးပြုခြင်းများ ထောက်လှမ်းနိုင်ရမည်
- **Regional Leaderboards:** Country/Region-level rankings ပြသနိုင်ရမည်
- **Notifications:** Rank ပြောင်းလဲပါက push notification ပေးပို့နိုင်ရမည်

### Non-Functional Requirements

| သတ်မှတ်ချက် | ပမာဏ |
|---|---|
| Concurrent Players | ၁၀၀ မီလီယံ (100M) |
| Score Update Throughput | ၁ မီလီယံ updates/second |
| Leaderboard Read Latency | ၅ms အောက် |
| Score Update → Display Latency | ၁ second အောက် (end-to-end) |
| Leaderboard Player Count | ၁ ဘီလီယံ+ |
| Game Event Replay Storage | ၆ လ |

### Scale Estimation

```
Players: 100 Million concurrent
Score updates/player/minute: 10 events average
Total score updates: 100M × 10 ÷ 60 = ~16.7M events/second

Leaderboard reads:
  Top 100 reads: 10M requests/second (very hot data)
  Own rank lookup: 1M requests/second

Storage:
  Score record: user_id(8B) + score(8B) + timestamp(8B) = 24 bytes
  1B players × 24 bytes = 24 GB (leaderboard, fits in Redis)
  Event replay: 1M events/sec × 100 bytes × 86400 = ~8.6 TB/day
```

---

## ၃၈.၂ Domain Decomposition: Service များ ခွဲခြားခြင်း

### ၁. Score Service
- Player score ingestion (WebSocket မှ score events လက်ခံသည်)
- Score validation ပြုလုပ်သည်
- Event stream Kafka သို့ publish ပြုလုပ်သည်
- Database: Cassandra (write-heavy, time-series)

### ၂. Leaderboard Service
- Redis Sorted Sets ဖြင့် real-time ranking manage ပြုလုပ်သည်
- Top-N queries handle ပြုလုပ်သည်
- Player rank lookup ပြုလုပ်သည်
- Storage: Redis Cluster

### ၃. Notification Service
- Rank change notifications ပေးပို့သည်
- Milestone notifications (Top 100, Top 1000 ရောက်ပါက)
- FCM/APNs integration

### ၄. Replay Service
- Game events ကို time-ordered sequence ဖြင့် သိမ်းဆည်းသည်
- Replay endpoint ပံ့ပိုးသည်
- Storage: S3 + Kafka (event sourcing)

### ၅. Anti-Cheat Service
- Score anomaly detection ပြုလုပ်သည်
- Velocity checks (score too fast)
- Statistical outlier detection
- ML model-based detection

---

## ၃၈.၃ Real-Time Score Ingestion

### WebSocket → Kafka Pipeline

```
Game Client (Score Event)
       │
       ▼ WebSocket
Score Ingestion Service (Stateless)
       │
       ├── Validate (auth token, game session)
       ├── Deduplicate (event_id check, Redis)
       │
       ▼ Publish
Kafka Topic: "score-events"
  Partition Key: user_id (same user → same partition → ordering)
```

### Score Event Format

```json
{
  "event_id": "uuid-xxx",
  "user_id": "player_123",
  "game_id": "match_456",
  "score_delta": 150,
  "score_total": 45820,
  "timestamp": 1712345678900,
  "action_type": "KILL",
  "metadata": {
    "weapon": "rifle",
    "location": "zone_3"
  }
}
```

### Kafka Consumer Groups

```
Kafka "score-events" topic
       │
       ├──▶ Consumer Group 1: Leaderboard Updater
       │    (Update Redis sorted sets)
       │
       ├──▶ Consumer Group 2: Replay Recorder
       │    (Persist to Cassandra/S3)
       │
       ├──▶ Consumer Group 3: Anti-Cheat Detector
       │    (Real-time anomaly analysis)
       │
       └──▶ Consumer Group 4: Notification Trigger
            (Check rank thresholds)
```

---

## ၃၈.၄ Leaderboard Design

### Redis Sorted Sets for Top-N Queries

Redis Sorted Set (ZSet) သည် leaderboard implementation ၏ core ဖြစ်သည်-

```
Redis ZADD Command:
ZADD leaderboard:global 45820 "player_123"
ZADD leaderboard:global 52000 "player_456"

Top 100 Query (O(log N + K)):
ZREVRANGE leaderboard:global 0 99 WITHSCORES

Player Rank Query (O(log N)):
ZREVRANK leaderboard:global "player_123"
→ Returns: 4523890 (0-indexed, so rank = 4,523,891)

Score Update:
ZADD leaderboard:global 46000 "player_123"  (atomic upsert)
```

### Multiple Leaderboard Types

```
leaderboard:global              → ကမ္ဘာလုံးဆိုင်ရာ ranking
leaderboard:country:MM          → Myanmar players ranking
leaderboard:game:battle_royale  → Game-specific ranking
leaderboard:weekly:2024-W03     → Weekly ranking (TTL: 8 days)
leaderboard:friends:player_123  → Friends ranking
```

### Approximate Rank for Billions of Players

**Challenge:** ၁ ဘီလီယံ+ players ရှိပါက Redis single sorted set သည် ~24 GB ဖြစ်ပြီး memory constraint ဖြစ်နိုင်သည်။

**Solution 1: Redis Cluster Sharding**
```
leaderboard:shard:0  → player IDs hash 0-3276799 (3.27M)
leaderboard:shard:1  → player IDs hash 3276800-6553599
...
leaderboard:shard:N  → last shard

Global rank = sum of ranks across shards
```

**Solution 2: Score Bucketing (Approximate Rank)**
```
Score range: 0 - 1,000,000
Buckets: 1000 buckets of size 1000 each

Bucket counts stored in Redis:
bucket:0-999    → 50,000 players
bucket:1000-1999 → 120,000 players
...

Player score = 45,820 → bucket 45
Approximate rank = sum(bucket counts 46 to max) + rank within bucket 45
Error margin: ±1000 players (acceptable for billions)
```

**Solution 3: HyperLogLog for Approximate Distinct Counts**
```
ကိုယ်ပိုင် rank ကို exactly မသိဘဲ approximate percentage ကို ပြသနိုင်သည်
"You are in top 5% of players"
```

---

## ၃၈.၅ Fan-Out Updates to Subscribers

### SSE / WebSocket Fan-Out

Score update ကို relevant subscribers များသို့ broadcast ပြုလုပ်ခြင်း-

```
Score Update Event (Kafka)
        │
        ▼
Fan-Out Service
        │
        ├── Type 1: Global Leaderboard Watchers
        │   (Top 100 page ကြည့်နေသူများ)
        │   → Redis Pub/Sub: "leaderboard:top100:update"
        │
        ├── Type 2: Friend Leaderboard Watchers
        │   → Per-user subscription channel
        │
        └── Type 3: Game Match Spectators
            → Match-specific channel
                    │
                    ▼
        WebSocket / SSE Connection Pool
        (50,000 connections per server)
                    │
                    ▼
              Client Apps
```

### Subscriber Management

```
User subscribes to leaderboard updates:
  WS Connect → Subscribe to:
    - "lb:global:top100" (if viewing top 100)
    - "lb:friends:{user_id}" (always)
    - "lb:match:{match_id}" (if in active match)

Update arrives:
  Score change → recalculate top 100
  If top 100 changed → publish to "lb:global:top100"
  Subscribers receive update without polling
```

**Throttling:** Top 100 leaderboard update ကို max ၁ second တစ်ကြိမ် throttle ပြုလုပ်သည် (client-side flicker မဖြစ်ရန်)

---

## ၃၈.၆ Anti-Cheat Detection

### Event Anomaly Detection

**Rule-Based Detection:**

```
Rule 1: Score Velocity Check
  Normal: max 1000 points/second per player
  Anomaly: > 5000 points/second → FLAG

Rule 2: Session Duration Check
  Normal: max 8 hours continuous play
  Anomaly: 24+ hours without break → FLAG

Rule 3: Statistical Outlier
  Player's average score: 1000-2000/game
  Sudden: 50,000/game → FLAG (25x deviation)

Rule 4: Bot Pattern Detection
  Human click variance: 150-300ms between actions
  Bot: exactly 100ms (perfect timing) → FLAG
```

**ML-Based Detection:**

```
Feature Engineering:
  - Score per minute distribution
  - Action timing variance
  - Win/loss ratio over time
  - Movement patterns (FPS games)
  - Network packet timing

Model: Isolation Forest / LSTM Anomaly Detection
Output: anomaly_score (0-1)
Threshold: > 0.8 → manual review
```

### Cheat Event Flow

```
Anti-Cheat Service detects anomaly
        │
        ▼
Publish to Kafka: "cheat-events"
        │
        ├──▶ Shadow-ban (hide from leaderboard)
        ├──▶ Alert Moderation Team
        └──▶ Freeze score (pending review)
```

---

## ၃၈.၇ Architecture Diagram နှင့် Data Flow

### Full System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                   GAME CLIENTS                                   │
│              (Mobile / PC / Console)                             │
└──────────┬──────────────────────────────────────────────────────┘
           │ WebSocket (Score Events)
           ▼
┌──────────────────────────────────────────────────────────────────┐
│              SCORE INGESTION SERVICE (Stateless)                 │
│           Auth Validation | Event Deduplication                  │
└──────────────────────┬───────────────────────────────────────────┘
                       │ Publish
                       ▼
              ┌─────────────────┐
              │      KAFKA      │
              │ "score-events"  │
              └────────┬────────┘
           ┌───────────┼───────────┬────────────┐
           ▼           ▼           ▼            ▼
    ┌────────────┐ ┌────────┐ ┌────────┐ ┌──────────┐
    │Leaderboard │ │Replay  │ │Anti-   │ │Notif.    │
    │  Updater   │ │Recorder│ │Cheat   │ │ Trigger  │
    └─────┬──────┘ └───┬────┘ └────────┘ └────┬─────┘
          │             │                      │
          ▼             ▼                      ▼
    ┌──────────┐  ┌──────────┐          ┌──────────┐
    │  Redis   │  │Cassandra │          │  FCM /   │
    │(ZSet     │  │(Event    │          │  APNs    │
    │Leaderbd) │  │ Replay)  │          │          │
    └─────┬────┘  └──────────┘          └──────────┘
          │
          ▼
    ┌──────────────────────────┐
    │   FAN-OUT SERVICE        │
    │   (WebSocket / SSE)      │
    └──────────────────────────┘
          │
          ▼
    ┌─────────────────────────────────────────┐
    │  CLIENT DISPLAY (Real-Time Leaderboard) │
    └─────────────────────────────────────────┘
```

### Score Update Data Flow (End-to-End)

```
Time 0ms:    Player kills enemy → client generates score event
Time 5ms:    Score event arrives at Ingestion Service
Time 6ms:    Event validated and published to Kafka
Time 10ms:   Leaderboard Updater consumes event
Time 11ms:   ZADD leaderboard:global 46000 "player_123" (Redis)
Time 12ms:   Check if top 100 changed → YES
Time 13ms:   Redis Pub/Sub: publish top100 update
Time 15ms:   Fan-Out Service receives update
Time 16ms:   Broadcast via WebSocket to 50M subscribers
Time 50ms:   Client display updates (including re-render)

Total E2E latency: ~50ms
```

---

## Key Design Decisions နှင့် Trade-offs

| Design Decision | ရွေးချယ်မှု | ကြောင်းရင်း |
|---|---|---|
| **Leaderboard Storage** | Redis Sorted Sets | O(log N) rank query, in-memory speed |
| **Score Ingestion** | Kafka | High throughput, consumer group isolation |
| **Rank for Billions** | Score Bucketing (Approximate) | Exact rank = too expensive at 1B+ scale |
| **Fan-Out** | WebSocket + Redis Pub/Sub | < 50ms end-to-end latency |
| **Anti-Cheat** | Rules + ML hybrid | Rules = fast, ML = complex patterns |
| **Replay Storage** | Kafka + Cassandra | Event sourcing for time-ordered replay |

### ပဓာန Trade-offs

- **Exact vs Approximate Rank:** ၁ ဘီလီယံ players အတွက် exact rank ကို O(log N) ဖြင့် Redis မှ ရနိုင်သော်လည်း memory GB ပေါင်းများစွာ လိုအပ်သည်။ Score bucketing ဖြင့် memory ၉၀% လျှော့ချနိုင်ပြီး "Top 5%" ကဲ့သို့ approximate display ပြုလုပ်နိုင်သည်

- **Consistency vs Availability:** Leaderboard update ကို eventual consistent ဖြင့် accept ပြုလုပ်သည်။ Network partition ဖြစ်ပါက player ၏ score update အချို့ delayed ဖြစ်နိုင်သော်လည်း eventually correct ဖြစ်မည်

- **Storage Cost vs Replay Fidelity:** Full event replay ကို Cassandra ထဲ ၆ လ သိမ်းဆည်းခြင်းသည် storage cost မြင့်သော်လည်း esports analysis, cheat investigation, highlight generation အတွက် မရှိမဖြစ် လိုအပ်သည်

- **Real-Time vs Batch Leaderboard:** Real-time leaderboard (Redis) သည် infrastructure cost မြင့်သော်လည်း player engagement ကို သိသိသာသာ တိုးမြင့်စေသည် (gamification effect)
