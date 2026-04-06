# အခန်း ၃၉: Video Streaming System (Netflix ကဲ့သို့သော Platform) ဒီဇိုင်း

## နိဒါန်း

Netflix သည် ကမ္ဘာပေါ်ရှိ အကြီးဆုံး video streaming platform ဖြစ်ပြီး နိုင်ငံ ၁၉၀ ကျော်တွင် user ၂၀၀ သန်းကျော်ကို serve လုပ်နေသည်။ 4K adaptive streaming, global CDN, per-title encoding optimization နှင့် personalized recommendation engine တို့သည် ဤ platform ၏ engineering marvel များဖြစ်သည်။ ဤအခန်းတွင် video ingestion pipeline, Open Connect CDN strategy, ABR algorithm, recommendation engine (batch + real-time hybrid) နှင့် DRM content protection တို့ကို microservices architecture ဖြင့် မည်သို့ ဒီဇိုင်းရေးဆွဲမည်ကို အသေးစိတ် လေ့လာမည်။

---

## ၃၉.၁ လိုအပ်ချက်များနှင့် Requirements အနှစ်ချုပ်

### Functional Requirements

- **Video Playback:** HD, 4K, HDR quality ဖြင့် smooth playback ပြုလုပ်နိုင်ရမည်
- **Adaptive Bitrate Streaming (ABR):** Network condition ပေါ်မူတည်၍ quality automatic ချိန်ညှိနိုင်ရမည်
- **Content Discovery:** Browse, search, recommendation feature ပံ့ပိုးနိုင်ရမည်
- **User Profiles:** Multiple profile per account, parental controls
- **Offline Downloads:** Mobile app ဖြင့် video download ပြုလုပ်နိုင်ရမည်
- **Resume Playback:** မဆုံးမချင်း ရပ်ထားသည့် နေရာမှ ဆက်ကြည့်နိုင်ရမည်
- **DRM Protection:** Piracy ကင်းစင်သော content delivery
- **Subtitles/Captions:** Multiple language support

### Non-Functional Requirements

| သတ်မှတ်ချက် | ပမာဏ |
|---|---|
| Total Users | ၂ ဘီလီယံ+ |
| Daily Active Users (DAU) | ၂၀၀ မီလီယံ |
| Concurrent Streams | ၈၀ မီလီယံ (peak) |
| Video Startup Time | ၂ second အောက် |
| Video Upload | ၅၀၀ hours/minute |
| Content Library | ၁ PB+ |
| CDN Cache Hit Ratio | ၉၅%+ |
| Global Availability | ၁၉၀+ နိုင်ငံ |

### Scale Estimation

```
Concurrent streams: 80M
Average video bitrate: 5 Mbps (HD streaming)
Total bandwidth: 80M × 5 Mbps = 400 Tbps

CDN edge servers needed:
  400 Tbps ÷ 10 Gbps per server = 40,000 CDN servers

Storage:
  1 hour video = 10 GB (multi-resolution)
  500 hours uploaded/min × 60 × 24 = 720,000 hours/day
  720,000 × 10 GB = 7.2 PB/day (new uploads)
  Long-tail catalog: 5 PB total
```

---

## ၃၉.၂ Domain Decomposition: Service များ ခွဲခြားခြင်း

### ၁. Ingestion Service
- Creator/Studio မှ raw video upload လက်ခံသည်
- Upload validation ပြုလုပ်သည်
- Transcoding queue သို့ submit ပြုလုပ်သည်

### ၂. Transcoding Service
- Multi-resolution encoding ပြုလုပ်သည် (240p, 480p, 720p, 1080p, 4K)
- HLS/DASH segment generation ပြုလုပ်သည်
- Subtitle extraction/embedding ပြုလုပ်သည်

### ၃. Metadata Service
- Video title, description, cast, genre သိမ်းဆည်းသည်
- Search index update ပြုလုပ်သည်
- Database: PostgreSQL + Elasticsearch

### ₄. Playback Service
- Stream URL generation ပြုလုပ်သည်
- DRM license request handle ပြုလုပ်သည်
- CDN redirect ပြုလုပ်သည်

### ₅. Recommendation Service
- Personalized recommendation generate ပြုလုပ်သည်
- Collaborative filtering + NLP model
- Real-time + batch pipeline

### ₆. Search Service
- Full-text search ပြုလုပ်သည်
- Elasticsearch-based

### ₇. Notification Service
- New content alerts ပေးပို့သည်
- Reminder notifications

### ₈. Billing Service
- Subscription management
- Payment processing

---

## ၃၉.၃ Video Ingestion နှင့် Transcoding Pipeline

### Multi-Resolution Encoding

Netflix တွင် content တစ်ခုကို **multiple resolutions နှင့် bitrates** ဖြင့် encode ပြုလုပ်သည်-

```
Raw Video (4K, 50GB)
        │
        ▼ Upload to S3 (Raw Bucket)
Transcoding Orchestrator (Kafka)
        │
        ├── Resolution Profiles:
        │   ┌──────────────────────────────────────┐
        │   │ Profile  │ Resolution │ Bitrate       │
        │   │──────────│────────────│───────────────│
        │   │ 240p     │ 426×240    │ 0.3 Mbps      │
        │   │ 480p     │ 854×480    │ 1.5 Mbps      │
        │   │ 720p     │ 1280×720   │ 3.0 Mbps      │
        │   │ 1080p    │ 1920×1080  │ 5.0 Mbps      │
        │   │ 4K HDR   │ 3840×2160  │ 15-25 Mbps    │
        │   └──────────────────────────────────────┘
        │
        ├── Parallel Transcoding Workers (GPU Instances)
        │   Worker 1 → 240p + 480p
        │   Worker 2 → 720p + 1080p
        │   Worker 3 → 4K HDR
        │   Worker 4 → Audio tracks (5.1 surround)
        │   Worker 5 → Subtitles processing
        │
        ▼
Segmented Output (HLS/DASH segments, 6 seconds each)
→ Upload to S3 (CDN Origin)
```

### HLS Manifest Structure

```
master.m3u8 (Master Playlist)
  │
  ├── 240p/playlist.m3u8
  │     ├── segment_000.ts
  │     ├── segment_001.ts
  │     └── ...
  │
  ├── 1080p/playlist.m3u8
  │     ├── segment_000.ts (6 seconds each)
  │     ├── segment_001.ts
  │     └── ...
  │
  └── 4k/playlist.m3u8
        └── ...
```

### Per-Title Encoding Optimization

Netflix ၏ နာမည်ကြီး innovation တစ်ခုဖြစ်သည်-

**Traditional:** Fixed bitrate-resolution profiles (same for all content)

**Per-Title Encoding:** ကာတွန်း (simple scenes) ကို ပုံမှန် bitrate ထက် ၅၀% လျှော့ encode ပြုလုပ်နိုင်ပြီး animation-heavy film ကို higher bitrate ဖြင့် encode ပြုလုပ်သည်-

```
Cartoon Movie:
  Standard 1080p: 5 Mbps → same quality
  Per-Title 1080p: 2.5 Mbps → same quality
  Saving: 50% bandwidth

Action Movie (explosions, fast motion):
  Standard 1080p: 5 Mbps → visible artifacts
  Per-Title 1080p: 8 Mbps → artifact-free
```

---

## ၃၉.၄ Content Delivery Network Strategy

### Open Connect Appliances (OCA)

Netflix ၏ ကိုယ်ပိုင် CDN ကို **Open Connect** ဟုခေါ်သည်-

```
Netflix HQ (Content Source)
        │
        ▼
Regional CDN Hub (PoP - Point of Presence)
e.g., Singapore PoP, Tokyo PoP, Frankfurt PoP
        │
        ▼
ISP-deployed OCAs (Open Connect Appliances)
e.g., Myanmar ISP သို့ hardware appliance ပေးပို့ deploy
        │
        ▼
End User (Direct from ISP OCA, low latency)
```

**Benefit:** User ၏ ISP ထဲပင် Netflix content cache ရှိသောကြောင့် latency အနည်းဆုံးဖြင့် stream လုပ်နိုင်သည်

### Cache Hit Ratio Optimization

**Popularity-Based Pre-warming:**
```
Daily batch job:
  Analyze: ပြသည်မည်ဆိုသော trending content
  Pre-push: Top 1000 titles ကို all edge OCAs သို့ proactive cache

Long-tail content:
  Demand-based caching (first request → origin → cache)
  LRU eviction policy
```

**Target: 95% cache hit ratio**
```
95% of requests → served from edge OCA
5% of requests → need to fetch from origin (S3)
```

---

## ၃၉.၅ Adaptive Bitrate Streaming (ABR)

ABR သည် network bandwidth ပေါ်မူတည်၍ video quality ကို automatic ချိန်ညှိပေးသည်-

```
Client Player State Machine:

  ┌──────────────────────────────────┐
  │  Network Monitor                 │
  │  - Measures throughput every 2s  │
  │  - Buffer level tracking         │
  └──────────────┬───────────────────┘
                 │
                 ▼
  ┌──────────────────────────────────┐
  │  ABR Algorithm                   │
  │                                  │
  │  if bandwidth > 15 Mbps:         │
  │    select 4K                     │
  │  elif bandwidth > 5 Mbps:        │
  │    select 1080p                  │
  │  elif bandwidth > 3 Mbps:        │
  │    select 720p                   │
  │  elif bandwidth > 1.5 Mbps:      │
  │    select 480p                   │
  │  else:                           │
  │    select 240p                   │
  └──────────────┬───────────────────┘
                 │
                 ▼
       Request next segment
       from appropriate quality
```

**Buffer Strategy:**
- Minimum buffer: 30 seconds (rebuffering ကာကွယ်ရန်)
- Maximum buffer: 3 minutes (memory constraint)
- Pre-fetch: Current quality + 1 quality level lower (fallback)

---

## ၃၉.၆ Playback Session နှင့် Resume Position Service

### Resume Position (Watch Progress)

```
User watches 45 minutes of movie, closes app:

Client → Playback Service:
POST /playback/position
{
  user_id: "u123",
  content_id: "movie_456",
  profile_id: "profile_1",
  position_seconds: 2700,  // 45 minutes
  timestamp: now()
}

Playback Service → Redis (write-through):
  SET "position:u123:movie_456:profile_1" "2700"
  (TTL: 180 days)

On next open:
GET /playback/position?user_id=u123&content_id=movie_456
→ Returns: {position: 2700, percentage: 45}
```

### Session State Management

```
Active Playback Session:
{
  session_id: "sess_abc",
  user_id: "u123",
  device_id: "iphone_xyz",
  content_id: "movie_456",
  quality: "1080p",
  drm_token: "...",
  started_at: timestamp,
  last_heartbeat: timestamp,
  cdn_server: "oca-sg-123"
}

Stored in Redis (TTL: 4 hours)
```

---

## ၃၉.၇ Recommendation Engine

### Collaborative Filtering + NLP

**Collaborative Filtering:** "You watched X, users who watched X also watched Y → recommend Y"

```
User-Item Matrix (sparse):
          Movie A  Movie B  Movie C  Movie D
User 1:     5       ?        3        ?
User 2:     ?       4        ?        5
User 3:     4       ?        ?        4

Matrix Factorization (ALS algorithm):
  Decompose → User Embeddings (U) × Item Embeddings (V)
  Predict missing values
  Recommend top-K highest predicted scores
```

**NLP on Watch History:**
- Video title, genre, description ကို NLP embedding ဖြင့် represent ပြုလုပ်သည်
- User watch history ကို "content-based" preference vector ဖြင့် မှတ်ယူသည်
- Semantic similarity ဖြင့် similar content recommend ပြုလုပ်သည်

### Real-Time vs Nightly Batch Pipeline

```
Real-Time Pipeline (< 1 minute):
  User watches movie → Kafka event → Flink streaming job
  → Update user's "recent interests" feature
  → Real-time recommendation refresh (not full retrain)

Nightly Batch Pipeline:
  Spark job runs at 2 AM
  → Full collaborative filtering model retrain
  → All user recommendations precomputed
  → Store in Redis (user_id → [rec1, rec2, ..., rec50])
  → Serve from Redis (< 5ms latency)

Hybrid:
  Base recommendations: nightly batch (Redis)
  Recent behavior adjustment: real-time feature update
```

---

## ၃၉.၈ A/B Testing — Artwork Thumbnails

Netflix ၏ thumbnail optimization သည် click-through rate ကို ၂x-၃x တိုးနိုင်သည်-

```
A/B Test Setup:
  Content: "Stranger Things"
  Variant A (30% traffic): Character-focused thumbnail
  Variant B (30% traffic): Monster-focused thumbnail
  Variant C (30% traffic): Action scene thumbnail
  Control (10% traffic): Original artwork

Metrics:
  - Click-through rate (CTR)
  - Play duration after click
  - Save to watchlist rate

Personalization Layer:
  User watches horror → show monster thumbnail
  User watches drama → show character thumbnail
  ML model predicts best thumbnail per user segment
```

---

## ၃၉.၉ DRM နှင့် Content Protection

### Widevine (Google) နှင့် FairPlay (Apple)

```
DRM Flow:
1. User requests to play content
2. Client → License Server: Request DRM license
   (includes device cert, content_id)
3. License Server validates:
   - User has active subscription
   - Device is authorized
   - Geographic restrictions OK
4. License Server → Client: Encrypted license key
5. Client decrypts and plays content

DRM Systems:
  Android/Chrome: Widevine L1/L2/L3
  iOS/Safari: FairPlay Streaming (FPS)
  Windows/Edge: PlayReady

Content Encryption:
  AES-128 encryption for HLS
  CENC (Common Encryption) for DASH
```

---

## ၃၉.၁၀ Architecture Diagram နှင့် Data Flow

### Full System Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                      CONTENT CREATORS                              │
│                   (Studios, Production Houses)                     │
└────────────────────────┬───────────────────────────────────────────┘
                         │ Raw Video Upload
                         ▼
┌────────────────────────────────────────────────────────────────────┐
│               INGESTION SERVICE + S3 (Raw Videos)                  │
└────────────────────────┬───────────────────────────────────────────┘
                         │ Transcoding Job
                         ▼
          ┌──────────────────────────────┐
          │    KAFKA (Job Queue)         │
          └──────────────┬───────────────┘
                         │
          ┌──────────────▼───────────────┐
          │   TRANSCODING WORKERS (GPU)  │
          │   Parallel: 240p/480p/720p   │
          │   1080p/4K, Audio, Subs      │
          └──────────────┬───────────────┘
                         │ HLS/DASH Segments
                         ▼
          ┌──────────────────────────────┐
          │   S3 CDN ORIGIN BUCKET       │
          └──────────────┬───────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │ OCA     │    │ OCA     │    │ OCA     │
    │ (Asia)  │    │(Europe) │    │(N.America)│
    └────┬────┘    └────┬────┘    └────┬────┘
         │              │              │
         └──────────────▼──────────────┘
                   End Users
                   (200M DAU)

─────────────────────────────────────────
APPLICATION TIER:
─────────────────────────────────────────
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Metadata │ │Playback  │ │Recommend │ │  Search  │
│ Service  │ │ Service  │ │ Service  │ │ Service  │
└────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
     │             │            │             │
     └─────────────▼────────────┘             │
              ┌──────────┐                    │
              │PostgreSQL│              ┌──────────┐
              │ (metadata│              │Elasticsrch│
              └──────────┘              └──────────┘
     ┌──────────────────────────────────────┐
     │          REDIS CLUSTER               │
     │  (Sessions, Positions, Rec Cache)    │
     └──────────────────────────────────────┘
```

### Video Playback Data Flow

```
1. User opens Netflix app, sees home page
   → Recommendation Service returns personalized list (from Redis cache)

2. User clicks "Play" on Movie X
   → Client → Playback Service: GET /play/movie_x
   → Playback Service checks subscription status
   → Playback Service → DRM License Server: get token

3. Playback Service returns:
   {
     manifest_url: "https://cdn.netflix.com/movie_x/master.m3u8",
     drm_token: "...",
     resume_position: 1800  // 30 min from last watch
   }

4. Client fetches master.m3u8 from nearest OCA
   → ABR algorithm selects 1080p (based on 12 Mbps bandwidth)
   → Client downloads segment_300.ts (30 min resume point)

5. Video plays, client sends heartbeat every 30s to Playback Service
   → Updates position in Redis

6. If bandwidth drops to 3 Mbps:
   → ABR switches to 720p seamlessly
   → Buffer prevents playback interruption
```

---

## Key Design Decisions နှင့် Trade-offs

| Design Decision | ရွေးချယ်မှု | ကြောင်းရင်း |
|---|---|---|
| **CDN** | Open Connect (own CDN) | ISP-level caching, control, cost savings |
| **Transcoding** | Parallel Workers + Kafka | Fast processing, fault-tolerant |
| **Per-Title Encoding** | Quality-based bitrate ladder | 50% bandwidth saving on simple content |
| **Recommendations** | Batch + Real-time Hybrid | Accuracy (batch) + Freshness (real-time) |
| **DRM** | Multi-DRM (Widevine+FairPlay) | Cross-platform support |
| **Resume Position** | Redis write-through | < 5ms read latency |

### ပဓာန Trade-offs

- **Storage vs Quality:** Per-title encoding ဖြင့် same perceptual quality ကို ပိုနည်းသော bits ဖြင့် deliver ပြုလုပ်နိုင်သော်လည်း encoding complexity (computation cost) တိုးသည်

- **CDN Cost vs Latency:** ကိုယ်ပိုင် CDN (Open Connect) တည်ဆောက်ခြင်းသည် capital expenditure မြင့်သော်လည်း long-term bandwidth cost နှင့် latency ကို သိသိသာသာ ကောင်းမွန်စေသည်

- **Recommendation Freshness vs Accuracy:** Real-time recommendation သည် fresh သော်လည်း model accuracy batch-trained model ထက် ကျဆင်းနိုင်သည်။ Hybrid approach ဖြင့် balance ညှိနိုင်သည်
