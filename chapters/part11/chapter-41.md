# အခန်း ၄၁: Video Upload နှင့် Sharing Platform ဒီဇိုင်း (YouTube ကဲ့သို့)

## နိဒါန်း

ဤအခန်းတွင် YouTube ကဲ့သို့သော ကမ္ဘာ့အကြီးဆုံး video platform တစ်ခုကို microservices architecture ဖြင့် ဒီဇိုင်းရေးဆွဲပုံကို အသေးစိတ် လေ့လာပါမည်။ တစ်မိနစ်လျှင် video ၅၀၀ နာရီစာ upload ဝင်လာခြင်း၊ billions of video views ကို serve လုပ်ခြင်း၊ real-time ad auction ပြုလုပ်ခြင်းတို့သည် ဤ system ၏ core challenges ဖြစ်ပါသည်။ Resumable upload, transcoding pipeline, fan-out strategies, approximate counters စသည့် engineering patterns များကို deep dive လုပ်ပါမည်။

## ၄၁.၁ Requirements နှင့် Scale Estimation

### Functional Requirements (လုပ်ဆောင်ချက်ဆိုင်ရာ လိုအပ်ချက်များ)

- **Video Upload**: Creator များ video upload ပြုလုပ်နိုင်ရမည် (minutes to hours length)
- **Video Playback**: adaptive bitrate streaming ဖြင့် smooth playback
- **Resumable Upload**: network failure ဖြစ်ပါက upload ကို resume ပြုလုပ်နိုင်ရမည်
- **Auto-Captions**: Speech-to-Text ဖြင့် subtitle အလိုအလျောက် ထုတ်ပေးခြင်း
- **Subscription System**: Channel subscribe လုပ်ခြင်း၊ new upload notifications
- **Comments**: Threaded comment system with moderation
- **Likes/Views**: Approximate counting ဖြင့် performance optimize
- **Ad Integration**: Pre-roll, mid-roll ad serving
- **Creator Analytics**: View count, watch time, revenue dashboard

### Non-Functional Requirements

- **Upload Rate**: တစ်မိနစ်လျှင် video **၅၀၀ နာရီ** upload ဝင်လာနိုင်ရမည်
- **Daily Views**: **၅ billion** video views/day ကို serve လုပ်နိုင်ရမည်
- **Upload-to-Available**: upload ပြီး **၅ မိနစ်** အတွင်း video ကြည့်နိုင်ရမည်
- **High Availability**: 99.9%+ uptime
- **Global Reach**: ကမ္ဘာတစ်ဝှမ်း CDN distribution

### Scale Estimation (အတိုင်းအတာ ခန့်မှန်းခြင်း)

```
Upload rate: 500 hours/min = 30,000 min video/minute
Raw file size: ~10 GB/hour average
Raw upload volume: 500 hours/min × 10 GB = ~5 TB/minute
Daily raw uploads: 5 TB × 60 × 24 = ~7.2 PB/day

Transcoded copies (5 resolutions + audio):
  ~7.2 PB × 1.5 (multi-resolution overhead) = ~10.8 PB/day total storage growth

Video view QPS: 5B/day / 86400 = ~57,870 QPS (peak 3x = 174K QPS)
Comment writes: ~1M/minute = 16,700 writes/second
10-year storage projection: ~39 EB (Exabytes) (with replication + backups)
```

## ၄၁.၂ Domain Decomposition (ဒိုမိန်းခွဲခြမ်းခြင်း)

1. **Upload Service** — Resumable chunked upload, chunk tracking (Redis), S3 multipart integration
2. **Transcoding Service** — Multi-resolution encoding, quality validation, virus scan, STT captions
3. **Metadata Service** — Video title, description, tags, category, search index (PostgreSQL + Elasticsearch)
4. **Comment Service** — Threaded comments, moderation pipeline (Cassandra for high write throughput)
5. **Like/Reaction Service** — Approximate counters, HyperLogLog for unique views (Redis)
6. **Subscription Service** — Channel subscriptions, fan-out for new uploads
7. **Feed Service** — Personalized video recommendations, ML ranking model
8. **Ad Service** — Real-Time Bidding (RTB), pre-roll/mid-roll insertion, revenue tracking

## ၄၁.၃ Resumable Video Upload

Large video files (GB-level) upload ပြုလုပ်ရာတွင် **Resumable Upload** protocol သည် မရှိမဖြစ် လိုအပ်ပါသည်။ Network interruption ဖြစ်ပါက ပြန်လည်ဆက်လက် upload လုပ်နိုင်ရပါမည်။

### Upload Flow (Upload လုပ်ငန်းစဉ်)

```
Step 1: Initiate Upload Session
  Client -> Upload Service:
  POST /upload/initiate
  {file_size: 5GB, filename: "video.mp4", content_type: "video/mp4"}
  Response: {upload_id: "up_xyz", chunk_size: 10MB}

Step 2: Upload Chunks (Parallel possible)
  For each 10MB chunk:
    PUT /upload/up_xyz/chunk/{chunk_number}
    Header: Content-Range: bytes 0-10485759/5368709120
    Body: <chunk_binary_data>
    Response: {chunk_number: 0, received: true, checksum: "md5..."}

Step 3: Complete Upload
  POST /upload/up_xyz/complete
  {total_chunks: 512, checksums: [...]}
  Response: {video_id: "vid_abc", status: "processing"}

Step 4: Resume (if interrupted)
  GET /upload/up_xyz/status
  Response: {uploaded_chunks: [0,1,2,3,5,6], missing: [4]}
  --> Client uploads only missing chunk #4
```

### Chunk Tracking (Redis)

```
Key: "upload:{upload_id}"
Value: {
  s3_multipart_id: "s3_mp_id",
  total_chunks: 512,
  uploaded_chunks: Set{0,1,2,...},
  status: "in_progress",
  created_at: timestamp
}
TTL: 7 days (incomplete uploads auto-expire)
```

S3 Multipart Upload API နှင့် integration ပြုလုပ်ပြီး chunk တစ်ခုချင်းစီကို S3 part upload အဖြစ် သိမ်းပါသည်။ အားလုံးပြီးစီးသောအခါ **CompleteMultipartUpload** API call ဖြင့် အပိုင်းများကို ပေါင်းစည်းပါသည်။

## ၄၁.၄ Transcoding Pipeline

Raw video ကို viewer devices အမျိုးမျိုးတွင် ဖွင့်ကြည့်နိုင်ရန် multiple resolutions နှင့် formats သို့ transcode လုပ်ရပါသည်။

### Pipeline Stages (Pipeline အဆင့်များ)

```
Raw Video (S3)
     |
     v  Kafka: "transcode-jobs"
+--------------------------------------------------+
|         TRANSCODING ORCHESTRATOR                  |
|                                                   |
| Stage 1: Quality Validation                       |
|   - Codec check (H.264/H.265/VP9)               |
|   - Resolution, frame rate, audio codec verify    |
|   - Reject corrupt/too-short files               |
|                                                   |
| Stage 2: Virus/Malware Scanning                   |
|   - ClamAV scan on raw file                      |
|   - Reject malicious files                        |
|                                                   |
| Stage 3: Content Policy Check (AI)                |
|   - Nudity detection (ML model)                  |
|   - Violence detection                            |
|   - Copyright match (Content ID fingerprint DB)   |
|                                                   |
| Stage 4: Speech-to-Text Captions                  |
|   - Whisper AI / Google Speech API               |
|   - Auto-generate .vtt subtitle file             |
|   - Multi-language translation (optional)         |
|                                                   |
| Stage 5: Multi-Resolution Transcoding (GPU)       |
|   - Parallel workers on GPU instances            |
|   - Output: 360p, 480p, 720p, 1080p, 4K         |
|   - Format: H.264 + H.265 + VP9                 |
|   - HLS segments (4-second chunks)               |
|                                                   |
| Stage 6: Thumbnail Extraction                     |
|   - Sample frames at 0%, 25%, 50%, 75%           |
|   - ML-selected best thumbnail (face + sharpness)|
+--------------------------------------------------+
     |
     v
S3 CDN Origin + CDN Distribution
     |
     v
Metadata Service: video status = "published"
```

**Content ID System** (မူပိုင်ခွင့် စစ်ဆေးခြင်း) သည် reference fingerprint DB (800M+ reference files) နှင့် upload video ၏ audio/video fingerprint ကို compare လုပ်ပါသည်။ Match တွေ့ပါက copyright owner ၏ policy (block/monetize/track) အရ ဆောင်ရွက်ပါသည်။

**Speech-to-Text** (STT) caption generation အတွက် OpenAI Whisper model ကို အသုံးပြုပြီး .vtt format subtitle file ကို auto-generate လုပ်ပါသည်။ English, Spanish, Hindi, Japanese စသည့် major languages အတွက် auto-translate ပါ ပြုလုပ်နိုင်ပါသည်။

Transcoding workers များသည် **GPU instances** (NVIDIA T4/A10G) ပေါ်တွင် run ပြီး Kafka-orchestrated parallel processing ဖြင့် 360p, 720p, 1080p, 4K resolutions ကို တစ်ပြိုင်နက် encode လုပ်ပါသည်။ HLS (HTTP Live Streaming) format ဖြင့် **4-second segments** အဖြစ် ခွဲပြီး adaptive bitrate streaming ပံ့ပိုးပါသည်။

## ၄၁.၅ Video Feed နှင့် Subscription System (Push vs Pull Fan-Out Hybrid)

New video upload ပြီးသောအခါ subscriber များ၏ feed ကို update လုပ်ရပါသည်။ Creator ၏ subscriber count ပေါ် မူတည်ပြီး strategy ကွဲပြားပါသည်:

### Push Fan-Out (Small/Medium Creators — < 1M subscribers)

```
New Video Event -> Fan-Out Service
  For each subscriber:
    LPUSH "feed:{subscriber_id}" "video_id"
    LTRIM "feed:{subscriber_id}" 0 999  // keep last 1000
  Optional: Push notification (FCM/APNs)
```

### Pull Fan-Out (Celebrity Creators — >= 1M subscribers)

Celebrity creator (ဥပမာ subscriber ၅၀M ရှိသူ) post လုပ်သောအခါ ၅၀M fan-out writes ပြုလုပ်ခြင်းသည် နာရီပေါင်းများစွာ ကြာနိုင်ပါသည်။ ထို့ကြောင့် **pull-on-read** strategy အသုံးပြုပါသည်:

```
User opens YouTube:
  Step 1: Read pre-built feed from Redis (pushed items)
  Step 2: Pull celebrity creators' recent videos separately
  Step 3: Merge + ML ranking sort
  Step 4: Return top 50 videos
```

### Hybrid Decision Logic

```
if subscriber_count < 1,000,000:
    strategy = PUSH  // fan-out on write
else:
    strategy = PULL  // fan-out on read

Feed = merge(pushed_feed_from_redis, pulled_celebrity_videos)
       .sort_by(ml_ranking_score)
       .limit(50)
```

## ၄၁.၆ Comment System at Scale (Threaded, Sorted, Moderated)

### Data Model (Cassandra)

```
Table: comments (partition key: video_id)
  video_id, comment_id (TimeUUID), parent_id, user_id,
  content, like_count, reply_count, created_at, is_deleted

Table: comment_replies (partition key: parent_comment_id)
  parent_comment_id, reply_id (TimeUUID),
  user_id, content, like_count, created_at
```

Cassandra ကို ရွေးချယ်ရခြင်းမှာ **high write throughput** (16K writes/second) လိုအပ်ပြီး video_id ကို partition key အဖြစ် သုံး၍ single video ၏ comments အားလုံးကို same partition တွင် co-locate လုပ်နိုင်သောကြောင့်ဖြစ်ပါသည်။

### Sorting Strategies (စီရီခြင်း နည်းလမ်းများ)

- **Top Comments**: Wilson score (statistical confidence-adjusted) = f(likes, replies, recency, reporter_ratio)။ ပုံမှန် precompute လုပ်ပြီး Redis cache တွင် သိမ်းပါသည်
- **Newest First**: Cassandra TimeUUID ordering (native, precompute မလို)
- **Pagination**: Cursor-based (comment_id as cursor) — offset-based ဖြစ်ပါက insert/delete ကြောင့် skip/duplicate ဖြစ်နိုင်သည်

### Comment Moderation Pipeline (မှတ်ချက် စစ်ဆေးခြင်း)

```
Comment Submit
     |
     +-> Sync Checks (< 100ms):
     |   - Spam text detection (ML classifier)
     |   - Profanity filter (word list + regex)
     |   - Link URL safety check
     |   - Rate limit (max 10 comments/minute/user)
     |
     +-> If passes: Post immediately (optimistic)
     |
     +-> Async Deep Check (Kafka consumer):
         - Hate speech ML model (BERT-based)
         - Coordinated inauthentic behavior detection
         - If flagged: shadow-ban or delete + notify
```

## ၄၁.၇ Like/View Count — Approximate Counters (HyperLogLog)

### HyperLogLog for Unique View Count (ထူးခြားသော ကြည့်ရှုသူ ရေတွက်ခြင်း)

Unique viewer count ကို exact counting ဖြင့် ရေတွက်ပါက memory အလွန်ကုန်ပါသည် (1M viewers x 8 bytes = 8MB per video)။ **HyperLogLog** (HLL) probabilistic data structure ဖြင့် memory ကို dramatically လျှော့ချပါသည်:

```
Redis HyperLogLog:
  PFADD "views:{video_id}" "{user_id}"    // add viewer
  PFCOUNT "views:{video_id}"               // get unique count

Memory: ~12 KB per video (regardless of cardinality)
Accuracy: 0.81% standard error
1 billion unique viewers = 12 KB (vs 8 GB exact counting)
```

### Like Count — Redis Counter with Background Sync

```
Real-time: Redis INCR "likes:{video_id}"
Background sync (every 30 seconds): Redis value -> PostgreSQL
Read path: Redis read (< 1ms)
Eventual consistency: DB may lag Redis by up to 30 seconds
```

### View Count Anomaly Detection (ကြည့်ရှုမှု ပုံမှန်မဟုတ်မှု စစ်ဆေးခြင်း)

Flink streaming job ဖြင့် fake views ကို စစ်ထုတ်ပါသည်:
- Same IP > 100 views/hour/video --> flag
- Watch duration < 3 seconds --> exclude
- Bot-like pattern detection (rhythmic viewing intervals)
- Initial count = raw views, 24 hours later = "Quality Views" (bot-filtered)

## ၄၁.၈ Ad Serving Integration (Pre-Roll, Mid-Roll)

### Real-Time Bidding (RTB) Flow

```
Video Play Request
     |
     v
Ad Decision Service (< 50ms total)
  1. User profile lookup (interests, demographics)
  2. Ad auction: request sent to multiple DSPs
     +-- Advertiser A: bid $2.50 CPM
     +-- Advertiser B: bid $3.10 CPM  <-- WINNER
     +-- Advertiser C: bid $1.80 CPM
  3. Winner determination: highest bid x quality score
  4. Return: {ad_url, skip_after: 5s, tracking_pixel}
```

### Ad Insertion Points

- **Pre-roll**: Video မစခင် (non-skippable <= 15s, skippable <= 30s)
- **Mid-roll**: Video ကြားထဲ (videos > 8 minutes) — creator marks ad_break timestamps (သို့) ML scene break detection
- **Post-roll**: Video ပြီးဆုံးချိန်
- **Overlay ads**: Video ပေါ်တွင် banner ပြသခြင်း

## ၄၁.၉ Architecture Diagram

```
+------------------------------------------------------------------+
|                    YOUTUBE CLIENTS                                 |
|           (Web, iOS, Android, TV, Gaming Consoles)                |
+-------+---------------------------+------------------------------+
        | Upload                    | View/Browse
        v                          v
+---------------+           +------------------+
| Upload Svc    |           | CDN Edge Nodes   |
| (Resumable    |           | (HLS segments)   |
| Chunks)       |           +--------+---------+
+-------+-------+                    |
        | Raw Video                  | Origin fetch (cache miss)
        v                           v
+---------------+           +------------------+
|  S3 (Raw)     |           | S3 CDN Origin    |
+-------+-------+           | (Transcoded)     |
        |                   +------------------+
        v
+----------------------------------------------+
|        TRANSCODING PIPELINE                   |
| Validate -> Scan -> Content ID -> STT        |
|  -> Multi-Res Encode (GPU) -> Thumbnails     |
| (Kafka-orchestrated, auto-scaling workers)    |
+----------------------------------------------+

============== APPLICATION SERVICES ================
+----------+ +----------+ +----------+ +----------+
| Metadata | | Comment  | | Like/    | |  Feed    |
| Service  | | Service  | | View Svc | | Service  |
+----+-----+ +----+-----+ +----+-----+ +----+-----+
     |            |            |             |
+----v-----+ +---v------+ +--v-------+ +---v------+
|Elastic-  | |Cassandra | | Redis    | | Redis    |
|search    | |(comments)| |(counters | |(feed     |
|(search)  | |          | | + HLL)   | | cache)   |
+----------+ +----------+ +----------+ +----------+

+----------+ +----------+
|Subscriptn| |   Ad     |
| Service  | | Service  |
+----+-----+ +----+-----+
     |            |
     v            v
  [KAFKA EVENT BUS]
  Topics: transcode-jobs | view-events | sub-events |
          comment-events | ad-impressions
     |
     +-> Fan-Out Worker -> Feed Updates
     +-> Analytics Worker -> Creator Dashboard
     +-> Ad Revenue Worker -> Payment Settlement
```

### Video Upload Complete Data Flow (Creator Perspective)

```
1. Creator opens YouTube Studio, selects 5GB video
2. Client: initiate upload -> upload_id
3. Divide into 10MB chunks, parallel upload 4 chunks simultaneously
4. Network failure at chunk #200 -> resume, upload missing chunks only
5. Upload complete -> S3 multipart merge -> Kafka "transcode-jobs"
6. Transcoding (< 5 minutes):
   Quality check -> Virus scan -> Content ID -> Auto-captions (Whisper)
   -> Transcode: 360p, 720p, 1080p parallel (GPU workers)
   -> Thumbnails: 3 options + ML best pick
7. CDN distribution: push to top 10 regional PoPs
8. Metadata Service: video status = "public"
9. Fan-out: < 1M subscribers -> push to feed cache
10. Creator notification: "Your video is live!"
    Total: upload complete to live = ~3-5 minutes
```

## Key Trade-offs (အဓိက ချိန်ညှိရမည့် အချက်များ)

| ချိန်ညှိချက် | ရွေးချယ်မှု | အကြောင်းပြချက် |
|---|---|---|
| **Upload** | Resumable chunked upload | GB-level files အတွက် network reliability မရှိမနေလိုအပ်သည် |
| **Comment Storage** | Cassandra | 16K writes/sec high throughput, time-series natural ordering |
| **View Count** | HyperLogLog (Redis) | 12KB per video ဖြင့် billions of unique viewers count နိုင်သည် (0.81% error acceptable) |
| **Feed Fan-Out** | Push-Pull Hybrid (1M threshold) | Small creators push (fast delivery), celebrities pull (avoid write storm) |
| **Transcoding** | Kafka-orchestrated parallel GPU | Stage failure ဖြစ်ပါက retry နိုင်ပြီး stages independent scale လုပ်နိုင်သည် |
| **Ad Decision** | RTB < 50ms | Real-time auction ဖြင့် revenue maximize ပြုလုပ်သော်လည်း latency budget tight ဖြစ်သည် |
| **Upload Speed vs Quality** | Initial publish 720p, background 4K | Creator ၅ မိနစ်အတွင်း live ဖြစ်ရမည့် SLA နှင့် transcoding quality balance |
| **Exact vs Approximate Count** | HLL approximate + daily exact reconciliation | Real-time UX အတွက် approximate (fast), revenue calculation အတွက် exact (batch) |

ဤ architecture တွင် transcoding pipeline သည် system ၏ အရေးကြီးဆုံး bottleneck ဖြစ်ပြီး GPU auto-scaling နှင့် Kafka-based orchestration ဖြင့် fault-tolerant, scalable pipeline တည်ဆောက်ထားပါသည်။ Resumable upload pattern သည် creator UX ကို dramatically improve လုပ်ပြီး HyperLogLog approximate counting သည် memory efficiency ကို 1000x ပိုကောင်းစေပါသည်။
