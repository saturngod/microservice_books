# အခန်း ၄၀: Music Streaming System ဒီဇိုင်း (Spotify ကဲ့သို့)

## နိဒါန်း

ဤအခန်းတွင် Spotify ကဲ့သို့သော ကမ္ဘာ့အဆင့် music streaming platform တစ်ခုကို microservices architecture ဖြင့် ဒီဇိုင်းရေးဆွဲပုံကို အသေးစိတ် လေ့လာပါမည်။ အသုံးပြုသူ သန်းရာနှင့်ချီကို real-time audio streaming ပေးပို့ရခြင်း၊ recommendation engine တည်ဆောက်ရခြင်း၊ royalty calculation တိကျစွာ ပြုလုပ်ရခြင်းတို့သည် ဤ system ၏ အဓိက စိန်ခေါ်မှုများ ဖြစ်ပါသည်။ Domain-Driven Design (DDD) နည်းလမ်းဖြင့် bounded contexts များ ခွဲခြမ်းပြီး service တစ်ခုချင်းစီ၏ internal architecture ကို deep dive လုပ်ပါမည်။

## ၄၀.၁ Requirements နှင့် Scale Estimation (လိုအပ်ချက်များနှင့် စနစ်အတိုင်းအတာ ခန့်မှန်းခြင်း)

### Functional Requirements (လုပ်ဆောင်ချက်ဆိုင်ရာ လိုအပ်ချက်များ)

- အသုံးပြုသူ **၆၀၀ သန်း** (600 million users) ကို ပံ့ပိုးနိုင်ရမည် — free tier နှင့် premium tier
- သီချင်း **၁၀၀ သန်းကျော်** (100M+ tracks) ကို catalog တွင် သိမ်းဆည်းနိုင်ရမည်
- **Podcast** (ပေါ့ဒ်ကတ်စ်) episode သန်းနှင့်ချီ ဖွင့်နိုင်ရမည်
- **Live Audio** (တိုက်ရိုက် အသံထုတ်လွှင့်ခြင်း) rooms ပံ့ပိုးရမည်
- **Playlist** (သီချင်းစာရင်း) ဖန်တီးခြင်း၊ collaborate လုပ်ခြင်း၊ မျှဝေခြင်း
- **Discover Weekly** — personalized recommendation playlist အပတ်စဉ် ထုတ်ပေးခြင်း
- **Artist Analytics** နှင့် **Royalty Calculation** — stream count အပေါ် မူတည်၍ မူပိုင်ခ တွက်ချက်ခြင်း
- **Social Features** — follow, friend activity, collaborative playlist
- **Offline Downloads** — premium users အတွက် encrypted offline playback

### Non-Functional Requirements (လုပ်ဆောင်ချက်မဟုတ်သော လိုအပ်ချက်များ)

- **Low Latency** (နှောင့်နှေးမှုနည်း): သီချင်းဖွင့်ရန် **200ms** အတွင်း audio playback စတင်နိုင်ရမည်
- **High Availability** (အမြဲရရှိနိုင်မှု): **99.99%** uptime ရှိရမည်
- **Horizontal Scalability** (ရေပြင်ညီ တိုးချဲ့နိုင်မှု): peak time တွင် concurrent streams **၅၀ သန်း** ပံ့ပိုးနိုင်ရမည်
- **Durability** (ခိုင်မြဲမှု): audio files ဘယ်တော့မှ ပျောက်ဆုံးခြင်းမရှိစေရ
- **Consistency** (ညီညွတ်မှု): royalty calculation အတွက် duplicate counting မဖြစ်စေရန် idempotent counting pipeline နှင့် reconciliation process ရှိရမည်

### Scale Estimation (အတိုင်းအတာ ခန့်မှန်းခြင်း)

```
Daily Active Users (DAU): 200M
Peak Concurrent Streams: 50M
Average track size: 5MB (OGG Vorbis 160kbps, 4-min track)
Total Audio Storage: 100M tracks x 5MB x 4 quality levels = 2 PB
Bandwidth at peak: 50M x 160kbps = 8 Tbps
Daily stream events: 200M DAU x 30 songs/day = 6 billion events/day
Playlist reads: 200M DAU x 10 reads/day = 2 billion reads/day
Playlist read QPS: 2B / 86400 = ~23,000 reads/second
Stream event QPS: 6B / 86400 = ~69,000 events/second
```

## ၄၀.၂ Domain Decomposition (ဒိုမိန်းခွဲခြမ်းခြင်း)

DDD (Domain-Driven Design) နည်းလမ်းဖြင့် system ကို bounded contexts (နယ်နိမိတ်သတ်မှတ်ထားသော ဆက်စပ်နယ်ပယ်များ) အဖြစ် ခွဲခြမ်းပါသည်:

1. **Catalog Service** — Track, Album, Artist metadata CRUD operations။ PostgreSQL + Elasticsearch (search index)
2. **Streaming Service** — Audio file delivery, adaptive bitrate selection, CDN coordination
3. **Playlist Service** — Playlist CRUD, collaborative playlist, read-heavy workload (Redis cache)
4. **Discovery Service** — Discover Weekly, Daily Mix, Release Radar recommendation algorithms
5. **Social Service** — Follow graph (Neo4j), friend activity feed, collaborative features
6. **Analytics Service** — Stream counting, artist dashboard, listener demographics (ClickHouse OLAP)
7. **Podcast Service** — RSS feed ingestion, episode management, live audio rooms
8. **User Service** — Authentication, authorization, subscription plan (free/premium/family)
9. **Search Service** — Full-text search, autocomplete, voice search (Elasticsearch)
10. **Ad Service** — Free tier audio/display ads serving, impression tracking

Service တစ်ခုချင်းစီသည် **Database per Service** pattern ဖြင့် ကိုယ်ပိုင် database ပိုင်ဆိုင်ပါသည်။ Inter-service communication (ဝန်ဆောင်မှုအချင်းချင်း ဆက်သွယ်ခြင်း) အတွက် synchronous calls (gRPC) နှင့် asynchronous events (Apache Kafka) နှစ်မျိုးလုံး အသုံးပြုပါသည်။ API Gateway (Kong/Envoy) မှ client requests ကို routing, rate limiting, authentication ပြုလုပ်ပါသည်။

## ၄၀.၃ Audio Ingestion နှင့် Transcoding Pipeline (အသံဖိုင်လက်ခံခြင်းနှင့် ပြောင်းလဲခြင်း)

Artist/label များက master audio file (WAV/FLAC) upload လုပ်သောအခါ ingestion pipeline စတင်ပါသည်:

### OGG Vorbis/AAC Multi-Bitrate Transcoding

```
Master Audio (WAV/FLAC, S3 upload)
     |
     v
[Ingestion Service] --> Validate format, sample rate, bit depth
     |
     v
[Audio Fingerprinting] --> Chromaprint algorithm
     |                     128-byte compact fingerprint ထုတ်ယူခြင်း
     v
[Deduplication Check] --> pgvector nearest neighbor search
     |                    similarity > 95% = duplicate flag
     v
[Loudness Normalization] --> EBU R128 standard, ReplayGain
     |
     v
[Transcode Job Queue (SQS/RabbitMQ)]
     |
     +-> Worker 1: OGG Vorbis 320kbps (Premium Very High)
     +-> Worker 2: OGG Vorbis 160kbps (Premium Standard)
     +-> Worker 3: AAC 96kbps (Free Normal)
     +-> Worker 4: AAC 24kbps (Mobile Data Saver)
     |
     v
[Completion Aggregator] --> Catalog DB update --> CDN pre-warm
```

**Audio Fingerprinting** (အသံလက်ဗွေ) သည် Chromaprint library ဖြင့် audio content ၏ unique identifier ထုတ်ယူပါသည်။ ဤ fingerprint ကို **Deduplication** (ထပ်တူရှာဖွေခြင်း) အတွက် PostgreSQL + pgvector extension တွင် nearest neighbor search ပြုလုပ်ပါသည်။ Copyright detection နှင့် cover song detection အတွက်လည်း အသုံးပြုပါသည်။

Transcoding workers များသည် **stateless Kubernetes pods** ဖြစ်ပြီး CPU-intensive workload အတွက် **spot instances** (ဈေးနှုန်းသက်သာသော compute instances) ကို အသုံးပြုပါသည်။ Fan-out pattern ဖြင့် သီချင်းတစ်ပုဒ်ကို bitrate ၄ မျိုး အပြိုင် encode လုပ်ပြီး completion aggregator က အားလုံးပြီးစီးမှ catalog ကို update လုပ်ပါသည်။

## ၄၀.၄ Streaming Delivery Architecture (သီချင်းပေးပို့ခြင်း ဗိသုကာ)

### Pre-Buffering (ကြိုတင်ကြားခံသိမ်းခြင်း)

Client app သည် user browsing ပြုလုပ်နေစဉ် playlist ၏ ထိပ်ဆုံးသီချင်း ၃ ပုဒ်၏ ပထမ ၁၅ စက္ကန့်ကို **prefetch** (ကြိုတင်ဆွဲယူ) လုပ်ထားပါသည်။ Play ခလုတ်နှိပ်သည်နှင့် audio သည် memory buffer ထဲတွင် ရှိပြီးသား ဖြစ်သောကြောင့် network round-trip မလိုအပ်ဘဲ **instant playback** (< 200ms) ရရှိပါသည်။ လက်ရှိသီချင်း၏ နောက်ဆုံး ၃၀ စက္ကန့်တွင် နောက်သီချင်းကို background download စတင်ပါသည်။

### Gapless Playback (ကြားဟခြင်းမရှိသော ဖွင့်ခြင်း)

- **HTTP Range Requests** ဖြင့် audio ကို 256KB chunks လိုက် download ပြုလုပ်ပါသည်
- Client audio decoder တွင် **crossfade buffer** ထားရှိ၍ track transition ချောမွေ့စွာ ဆောင်ရွက်ပါသည်
- **Adaptive Bitrate Streaming** — bandwidth estimation algorithm ဖြင့် network condition ပေါ်မူတည်ပြီး 320kbps မှ 96kbps သို့ seamless switch ပြုလုပ်ပါသည်
- Seek optimization — byte-range mapping table ကို track metadata တွင် embed လုပ်ပြီး **O(1) seek** ဆောင်ရွက်ပါသည်

### Offline Downloads with Encrypted Local Cache

Premium users အတွက် offline download feature ကို **encrypted local cache** ဖြင့် implement လုပ်ပါသည်:

- **AES-256-GCM encryption** ဖြင့် audio file ကို encrypt လုပ်ပြီး device storage တွင် သိမ်းပါသည်
- Decryption key ကို **secure enclave** (Android Keystore / iOS Keychain) တွင် သိမ်းပါသည်
- **License check** — device ၅ ခုထိ offline sync ခွင့်ပြုပြီး **၃၀ ရက်တစ်ကြိမ်** online verify ပြုလုပ်ရပါသည်
- **DRM integration** — Widevine (Android), FairPlay (iOS) content protection
- **LRU eviction** — disk space ပြည့်သည့်အခါ ကြာရှည်မဖွင့်သော tracks ကို အလိုအလျောက် ဖျက်ပါသည်

## ၄၀.၅ Playlist နှင့် Library Service (Read-Heavy with Redis Cache)

Playlist service သည် **read-heavy workload** ဖြစ်ပြီး write:read ratio 1:100 ရှိပါသည်။

### Data Model

```sql
-- PostgreSQL (sharded by owner_id)
playlists: id, owner_id, name, description, is_public, is_collaborative,
           follower_count, created_at, updated_at

playlist_tracks: playlist_id, track_id, position, added_at, added_by
```

### Multi-Layer Caching Strategy

```
Level 1: Client-side cache (device memory, TTL 30 min)
Level 2: CDN Edge Cache (public playlists like "Top 50", TTL 5 min)
Level 3: Redis Cluster (primary backend cache)
         - Playlist metadata: Redis Hash (HSET playlist:{id} name "...")
         - Track list: Redis Sorted Set (ZADD playlist:{id}:tracks {position} {track_id})
         - Cache invalidation: Write-through pattern
Level 4: PostgreSQL (Source of Truth)
```

Redis Cluster ကို primary cache အဖြစ် အသုံးပြုပြီး **write-through** pattern ဖြင့် DB write တိုင်း cache ကို synchronous update လုပ်ပါသည်။ Popular playlists (Discover Weekly, RapCaviar) အတွက် **CDN edge cache** ထပ်ထည့်ပြီး cache warming job ဖြင့် ကြိုတင် load လုပ်ပါသည်။ Database sharding key မှာ `owner_id` ဖြစ်ပြီး user တစ်ဦး၏ playlist အားလုံးကို **single shard** တွင် co-locate လုပ်ပါသည်။

## ၄၀.၆ Discover Weekly Recommendation Engine

Spotify ၏ ထင်ရှားသော feature ဖြစ်သည့် Discover Weekly သည် **Collaborative Filtering** (ပူးပေါင်းစစ်ထုတ်ခြင်း) နှင့် **NLP** (Natural Language Processing) ကို ပေါင်းစပ်ပါသည်။

### Collaborative Filtering

- **User-User CF** — နားဆင်မှုပုံစံတူသော users ရှာဖွေပြီး cross-recommend ပြုလုပ်ခြင်း (cosine similarity)
- **Item-Item CF** — playlist co-occurrence matrix ဖြင့် တွဲဖွင့်တတ်သော tracks ရှာဖွေခြင်း
- **Matrix Factorization** — ALS (Alternating Least Squares) algorithm ဖြင့် user-track matrix ကို 128-dimensional latent vectors အဖြစ် decompose လုပ်ခြင်း

### NLP on Playlist Names

User-created playlist names ("chill vibes", "workout energy", "sad boi hours") ကို **Word2Vec/BERT embedding** ဖြင့် vector representation အဖြစ် ပြောင်းပါသည်။ **LDA topic modeling** ဖြင့် playlist themes ရှာဖွေပြီး similar playlists ကြား cross-pollination recommendation ပေးပါသည်။

### Batch Pipeline (Hadoop/Spark + Feature Store)

```
[Listening Events, Skip/Save, Playlist Edits]
     |
     v
[Kafka] --> [Spark Streaming] --> Feature Store (Redis: online serving)
         --> [Spark Batch - Weekly] --> Feature Store (Cassandra: offline training)
                |
                v
         ALS Matrix Factorization + NLP Analysis + Audio Features
                |
                v
         Candidate Generation (top 500 tracks per user)
                |
                v
         Ranking Model (LightGBM / Neural Network)
                |
                v
         Final 30 Tracks --> Write to Playlist DB
                |
                v
         Push Notification: "Your Discover Weekly is ready"
```

**Feature Store** (အင်္ဂါရပ် သိုလှောင်ခန်း) သည် dual-layer storage ဖြစ်ပြီး Redis (real-time model serving) နှင့် Cassandra (batch model training) ကို ပေါင်းစပ်ပါသည်။ User features (listening history, skip rate, time-of-day preference) နှင့် track features (tempo, energy, danceability, valence) ကို သိမ်းပါသည်။

## ၄၀.၇ Artist Analytics နှင့် Royalty Calculation

### Stream Count Events Pipeline

```
[Client Play Event] --> [Event Collector: validate >30s, not bot]
     |
     v
[Kafka Topic: "validated-stream-events"]
     |
     +--> [Kafka Streams / Flink] --> Real-Time Artist Dashboard (ClickHouse)
     |
     +--> [Spark Batch - Monthly] --> Royalty Aggregation --> Settlement
     |
     +--> [Flink] --> Fraud Detection (bot stream filtering)
```

### Stream Validation (ဖွင့်ရှုမှု အတည်ပြုခြင်း)

- **Minimum play duration**: ၃၀ စက္ကန့်ကျော်မှ stream တစ်ကြိမ်အဖြစ် count ပါသည်
- **Bot detection**: IP clustering, device fingerprinting, listening pattern analysis ဖြင့် artificial streams စစ်ထုတ်ပါသည်
- **Exactly-once counting**: Kafka transactions + idempotency key (user_id + track_id + timestamp_bucket) ဖြင့် duplicate events ကို စစ်ပါသည်

### Royalty Settlement

**Pro-rata model** ဖြင့် monthly subscription revenue pool ကို total streams အချိုးကျ ခွဲဝေပါသည်:

```
Artist Payment = (Artist Streams / Total Platform Streams)
                 x Revenue Pool x (1 - Platform Fee)
```

Monthly batch job: ClickHouse aggregation --> royalty engine --> ledger DB --> payment gateway --> bank settlement။ Songwriter, producer, label, publisher အကြား contract terms အရ **multi-party split** ပြုလုပ်ပါသည်။

## ၄၀.၈ Podcast နှင့် Live Audio Pipeline

### Podcast Ingestion

- **RSS feed polling** (15-minute interval) ဖြင့် publisher RSS feed စစ်ဆေးပြီး episode အသစ် ဆွဲယူခြင်း
- **Speech-to-Text transcription** — OpenAI Whisper model ဖြင့် podcast audio ကို text ပြောင်းပြီး Elasticsearch index တွင် ထည့်ခြင်းဖြင့် content ကို keyword search လုပ်နိုင်ခြင်း
- **Dynamic Ad Insertion (DAI)** — listener demographics/location ပေါ်မူတည်ပြီး podcast stream ထဲသို့ targeted ad real-time ထည့်သွင်းခြင်း
- Chapter markers ဖြင့် episode sections သို့ seek လုပ်နိုင်ခြင်း

### Live Audio Rooms

- **WebRTC** ဖြင့် host audio ingestion
- **SFU (Selective Forwarding Unit)** — Janus/LiveKit ဖြင့် small rooms (< 10K) direct stream, large rooms HLS/DASH adaptive streaming
- **Recording** — session ကို S3 တွင် record လုပ်ပြီး podcast episode အဖြစ် ပြန်ဖွင့်နိုင်ခြင်း
- **Real-time chat** — Redis Pub/Sub + WebSocket

## ၄၀.၉ Architecture Diagram နှင့် Data Flow

```
+------------------------------------------------------------------+
|                     CLIENT APPS                                   |
|           (Mobile / Desktop / Web / Smart Speaker)                |
+------------------------------+-----------------------------------+
                               |
                        +------v------+
                        | API Gateway |
                        | (Kong/Envoy)|
                        | Rate Limit  |
                        | Auth / JWT  |
                        +------+------+
                               |
      +--------+--------+-----+-----+--------+--------+--------+
      |        |        |           |        |        |        |
  +---v--+ +--v---+ +--v----+ +---v---+ +--v---+ +--v----+ +-v------+
  |Catlg | |Strm  | |Play-  | |Disco- | |Socal | |Podcst | |Analytcs|
  |Svc   | |Svc   | |list   | |very   | |Svc   | |Svc    | |Svc     |
  +--+---+ +--+---+ +--+----+ +--+----+ +--+---+ +--+----+ +--+-----+
     |        |        |          |        |        |          |
  +--v---+ +--v---+ +--v----+ +--v----+ +-v----+ +-v-----+ +-v------+
  |PgSQL | | CDN  | |Redis  | |Feature| |Neo4j | |PgSQL  | |Click-  |
  |+ES   | |+S3   | |Cluster| |Store  | |Graph | |(pod   | |House   |
  |(meta)| |Bucket| |(cache)| |(Redis | |DB    | | meta) | |(OLAP)  |
  +------+ +------+ +---+---+ |+Cass) | +------+ +-------+ +--------+
                         |     +-------+
                     +---v-----------+
                     | PostgreSQL    |
                     | (Sharded by   |
                     |  owner_id)    |
                     +---------------+

=================== EVENT BUS ====================================
          [Apache Kafka Cluster - Event Backbone]
  Topics: stream-events | playlist-updates | user-actions |
          podcast-events | ad-impressions | social-events
==================================================================

  Ingestion Pipeline:
  Artist Upload -> S3 -> [Ingestion Svc] -> Fingerprint -> Dedup
    -> [SQS Transcode Queue] -> K8s Workers -> S3 (multi-bitrate)
    -> CDN Pre-warm -> Catalog DB Update

  Recommendation Pipeline:
  Kafka Events -> Spark Streaming -> Feature Store (real-time)
  Kafka Events -> Spark Batch (weekly) -> ALS + NLP -> Discover Weekly

  Royalty Pipeline:
  Validated Streams -> Kafka -> Flink (fraud detect)
    -> ClickHouse (aggregate) -> Monthly Settlement -> Payment
```

## Key Trade-offs (အဓိက ချိန်ညှိရမည့် အချက်များ)

| ချိန်ညှိချက် | ရွေးချယ်မှု | အကြောင်းပြချက် |
|---|---|---|
| **Storage Format** | OGG Vorbis + AAC | OGG သည် open format ဖြစ်ပြီး quality/size ratio ကောင်းသည်။ AAC သည် Apple ecosystem compatibility အတွက် မဖြစ်မနေ လိုအပ်သည် |
| **Cache Strategy** | Redis write-through | Read-heavy (1:100 ratio) workload အတွက် 99%+ cache hit rate ရရှိပြီး write-through ဖြင့် consistency ထိန်းသိမ်းသည် |
| **Recommendation** | Batch + Near-RT hybrid | Weekly batch (Spark) သည် deep model training ပြုလုပ်နိုင်ပြီး near-real-time (Kafka Streams) သည် skip/save signal ကို instantly reflect လုပ်နိုင်သည် |
| **Stream Counting** | Kafka transactions + idempotent sink + reconciliation | Royalty accuracy critical ဖြစ်ပြီး duplicate counting risk ကို stream processor တစ်ခုတည်းအပေါ် မထားဘဲ ledger-style reconciliation ဖြင့် ထိန်းသိမ်းရသည် |
| **Offline Encryption** | AES-256-GCM + license check | DRM compliance နှင့် UX balance — ၃၀ ရက်တစ်ကြိမ် verify သည် piracy ကာကွယ်ပြီး user ကို မထိခိုက်ပါ |
| **Social Graph** | Neo4j (Graph DB) | Follow relationship traversal (friend-of-friend) ကဲ့သို့ graph-native queries များတွင် relational DB ထက် ပိုသင့်တော်ပြီး query complexity ကို လျှော့ချနိုင်သည် |
| **Database Sharding** | User-based (owner_id) | Single shard co-location ဖြင့် cross-shard queries minimize လုပ်နိုင်သည် |
| **CDN Strategy** | Multi-CDN (CloudFront + Fastly) | Single CDN outage failover + ပိုကောင်းသော geographic coverage |

ဤ architecture သည် service တစ်ခုချင်းစီကို independent deploy နှင့် scale လုပ်နိုင်ပြီး Kafka event bus သည် loose coupling ဖန်တီးပေးကာ system resilience ကို မြှင့်တင်ပါသည်။ Feature Store pattern သည် ML training နှင့် online serving ကြား bridge အဖြစ် ဆောင်ရွက်ပြီး recommendation quality ကို continuous improvement ပြုလုပ်နိုင်စေပါသည်။
