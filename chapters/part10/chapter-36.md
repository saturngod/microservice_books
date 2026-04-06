# အခန်း ၃၆: Chat နှင့် Messaging System (WhatsApp / Slack စနစ်ဒီဇိုင်း)

## နိဒါန်း

Real-time messaging system သည် ကမ္ဘာပေါ်ရှိ လူ ဘီလီယံပေါင်းများစွာ နေ့စဉ် အသုံးပြုနေသော infrastructure ဖြစ်သည်။ WhatsApp သည် တစ်ရက်လျှင် message ၁၀၀ ဘီလီယံကျော် ကိုင်တွယ်ပြီး Slack သည် enterprise collaboration ၏ backbone ဖြစ်သည်။ ဤအခန်းတွင် WebSocket connection management, message delivery guarantees, end-to-end encryption, presence system နှင့် media delivery pipeline တို့ကို microservices architecture ဖြင့် မည်သို့ ဒီဇိုင်းရေးဆွဲမည်ကို အသေးစိတ် လေ့လာမည်။

---

## ၃၆.၁ လိုအပ်ချက်များနှင့် Requirements အနှစ်ချုပ်

### Functional Requirements

ကျွန်ုပ်တို့ တည်ဆောက်မည့် Chat System သည် အောက်ပါ လုပ်ဆောင်ချက်များကို ပံ့ပိုးရမည်-

- **တစ်ဦးချင်း မက်ဆေ့ပေးပို့ခြင်း (1-to-1 Messaging):** User နှစ်ဦးကြား real-time စကားပြောဆိုနိုင်ရမည်
- **Group Chat:** အများဆုံး User ၅၀၀ ပါဝင်သော Group မက်ဆေ့ပေးပို့နိုင်ရမည်
- **Media ပေးပို့ခြင်း:** ရုပ်ပုံ၊ ဗီဒီယို၊ ဖိုင်များ ပေးပို့နိုင်ရမည်
- **Offline Delivery:** User offline ဖြစ်နေချိန် မက်ဆေ့များ သိမ်းဆည်းပြီး ပြန်ဝင်လာချိန် deliver ပြုလုပ်ရမည်
- **Read Receipts:** မက်ဆေ့ကို ဖတ်ပြီးကြောင်း အသိပေးနိုင်ရမည်
- **End-to-End Encryption (E2EE):** မက်ဆေ့များကို Server မှ ဖတ်၍မရသော ပုံစံဖြင့် encrypt ပြုလုပ်ရမည်
- **Presence System:** User Online/Offline/Typing အခြေအနေ ပြသနိုင်ရမည်

### Non-Functional Requirements

| သတ်မှတ်ချက် | ပမာဏ |
|---|---|
| တစ်နေ့ မက်ဆေ့ပေးပို့ပမာဏ | ၁၀၀ ဘီလီယံ (100 Billion messages/day) |
| Active Users | ၂ ဘီလီယံ (2 Billion users) |
| Daily Active Users (DAU) | ၅၀၀ မီလီယံ |
| Message Delivery Latency | ၅၀ms အောက် (P99) |
| မက်ဆေ့ retention | ၁ နှစ် |
| Media ဖိုင် retention | ၃ လ |

### Scale Estimation

**QPS တွက်ချက်ခြင်း:**
```
100 Billion messages/day ÷ 86,400 seconds = ~1.16 Million messages/second
Peak QPS (3x) = ~3.5 Million messages/second
```

**Storage တွက်ချက်ခြင်း:**
```
Average message size = 100 bytes
100B messages/day × 100 bytes = 10TB/day
1 year retention = 10TB × 365 = ~3.65 PB (text only)

Media messages (20% of total):
20B media messages × 500KB average = ~10 PB/day
CDN + compression ဖြင့် 50% လျှော့ = ~5 PB/day
```

**Server Estimation:**
```
Assumption: Average user session = 30 minutes
500M DAU × 30 min average session ÷ (24 × 60 min/day) = ~10.4M average concurrent
Peak concurrent (2.5x average) = ~26M concurrent WebSocket connections

တစ်လုံး Server = 50,000 concurrent WebSocket connections ကို handle
26M ÷ 50,000 = ~520 WebSocket Servers (minimum)
Redundancy factor 2x = ~1,040 WebSocket Servers လိုအပ်
```

---

## ၃၆.၂ Domain Decomposition: Service များ ခွဲခြားခြင်း

Chat System ကို အောက်ပါ Microservices များအဖြစ် ခွဲခြားသည်-

### ၁. User Service
- User Registration, Authentication, Profile Management
- JWT Token ထုတ်ပေးခြင်း
- Database: PostgreSQL (User data)

### ၂. Connection Service (Session Service)
- WebSocket Connection များ manage ပြုလုပ်ခြင်း
- User ဘယ် WebSocket Server နှင့် ချိတ်ဆက်နေသည်ကို track လုပ်ခြင်း
- Redis ဖြင့် Connection Registry ထိန်းသိမ်းခြင်း

### ၃. Message Service
- မက်ဆေ့ send/receive ပြုလုပ်ခြင်း
- Message ID generation
- Duplicate detection
- Database: Cassandra (write-heavy workload အတွက်)

### ၄. Group Service
- Group ဖန်တီးခြင်း၊ Member စီမံခန့်ခွဲခြင်း
- Group metadata သိမ်းဆည်းခြင်း
- Database: PostgreSQL

### ၅. Notification Service
- Push Notification (APNs, FCM) ပေးပို့ခြင်း
- Offline User များအတွက် notification handle ပြုလုပ်ခြင်း

### ၆. Media Service
- ဖိုင် upload/download handle ပြုလုပ်ခြင်း
- Thumbnail generation
- Storage: Amazon S3 / Object Storage

### ၇. Presence Service
- User Online/Offline/Typing status track လုပ်ခြင်း
- Heartbeat mechanism
- Storage: Redis (in-memory, fast reads)

---

## ၃၆.၃ WebSocket Connection Management at Scale

### Session Server Architecture

WebSocket-based real-time messaging တွင် HTTP ကဲ့သို့ stateless မဟုတ်ဘဲ **stateful connection** ဖြစ်သည်။ ထို့ကြောင့် Session Server Management သည် အရေးကြီးသော challenge တစ်ခုဖြစ်သည်။

```
Client App
    │
    ▼ (WebSocket Upgrade Request)
Load Balancer (Layer 7)
    │
    ├──▶ WebSocket Server 1 (WS-1)
    ├──▶ WebSocket Server 2 (WS-2)
    ├──▶ WebSocket Server 3 (WS-3)
    └──▶ WebSocket Server N (WS-N)
         │
         ▼
    Redis Connection Registry
    {user_id: "WS-3", socket_id: "abc123"}
```

### Connection Registry with Redis

User တစ်ယောက် WebSocket Server သို့ connect လုပ်သောအခါ Redis ထဲ record တစ်ခု သွင်းသည်-

```
Key: connection:{user_id}
Value: {
  server_id: "ws-server-3",
  socket_id: "abc123",
  connected_at: 1712345678,
  ttl: 300  // heartbeat မပို့ပါက expire
}
```

မက်ဆေ့ routing ပြုလုပ်ရာတွင်-
1. Sender က မက်ဆေ့ပေးပို့သည်
2. Message Service က Redis မှ recipient ဘယ် Server နှင့် ချိတ်ဆက်နေသည် စစ်ဆေးသည်
3. ထို Server သို့ Internal Message Queue မှတဆင့် route ပြုလုပ်သည်

### Sticky Sessions vs Stateless Routing

| Approach | အကျိုးကျေးဇူး | အားနည်းချက် |
|---|---|---|
| **Sticky Sessions** | Simple, Load Balancer မှ manage | Server crash ဖြစ်ပါက reconnect ပြဿနာ |
| **Stateless Routing** | Fault-tolerant, horizontal scaling လွယ် | Redis lookup ကြောင့် latency တိုးနိုင် |

**ကျွန်ုပ်တို့ choose ပြုလုပ်မည့် approach:** Redis-based Stateless Routing ကို အသုံးပြုမည်။ ၎င်းသည် Server crash ဖြစ်သောအခါ Client အသစ် reconnect ပြုလုပ်ပြီး Redis record ကို update လုပ်သည်။

---

## ၃၆.၄ Message Delivery Guarantees

### At-Least-Once Delivery with Client ACK

မက်ဆေ့ မပျောက်ဆုံးစေရန် **At-Least-Once Delivery** model ကို အသုံးပြုသည်-

```
Sender                Message Service          Recipient
   │                        │                      │
   │─── Send Message ───────▶│                      │
   │                        │──── Deliver ─────────▶│
   │                        │◀─── ACK ─────────────│
   │◀─── Delivery ACK ──────│                      │
```

**ACK မရပါက:** Message Service သည် exponential backoff ဖြင့် retry ပြုလုပ်သည် (1s, 2s, 4s, 8s, max 60s)

### Message ID Deduplication

Network retry ကြောင့် မက်ဆေ့ မှားယွင်းစွာ ထပ်ရောက်ရာမှ ကာကွယ်ရန် **Client-generated Message ID** ကို အသုံးပြုသည်-

```
message_id = UUID-v4 (Client မှ generate)
```

Server သည် မက်ဆေ့ receive လုပ်သောအခါ `message_id` ကို Redis ထဲ check ပြုလုပ်ပြီး ထပ်နေပါက discard ပြုလုပ်သည်-

```
if Redis.exists("msg_dedup:{message_id}"):
    return "duplicate, ignored"
else:
    Redis.setex("msg_dedup:{message_id}", 86400)  // 24 hours TTL
    process_message()
```

### Offline Message Queue

User offline ဖြစ်နေပါက မက်ဆေ့ Cassandra ထဲ သိမ်းဆည်းထားပြီး ပြန်ဝင်လာချိန် deliver ပြုလုပ်သည်-

```
Table: offline_messages
- user_id
- message_id
- sender_id
- content (encrypted)
- created_at
- TTL: 30 days
```

---

## ၃၆.၅ Message Ordering in Group Chats

Group Chat တွင် မက်ဆေ့ ordering သည် distributed system challenge တစ်ခုဖြစ်သည်။

**Lamport Timestamp** ကို Server-side sequence number အဖြစ် အသုံးပြုသည်-

```
Group Chat Sequence:
- User A sends msg at T=1001 → Server assigns seq=1
- User B sends msg at T=1002 → Server assigns seq=2
- User A sends msg at T=1003 → Server assigns seq=3
```

Cassandra ထဲ သိမ်းဆည်းပုံ-
```
Table: group_messages
- group_id (partition key)
- sequence_number (clustering key, DESC)
- message_id
- sender_id
- content
- timestamp
```

---

## ၃၆.၆ End-to-End Encryption (Signal Protocol)

**Signal Protocol** သည် WhatsApp, Signal app တို့တွင် အသုံးပြုသော E2EE standard ဖြစ်သည်။

### Key Exchange Mechanism

```
Alice                          Server                         Bob
  │                              │                              │
  │──── Upload PreKeys ─────────▶│                              │
  │                              │◀──── Upload PreKeys ─────────│
  │──── Request Bob's PreKey ───▶│                              │
  │◀─── Bob's PreKey Bundle ─────│                              │
  │                              │                              │
  │═══ Double Ratchet Algorithm ═════════════════════════════▶  │
  │   (X3DH Key Agreement + Forward Secrecy)                    │
```

### Key Components

- **Identity Key (IK):** Long-term key pair — User identity ကိုယ်စားပြုသည်
- **Signed PreKey (SPK):** Medium-term key — Server မှ distribute ပြုလုပ်သည်
- **One-Time PreKey (OPK):** Single-use key — forward secrecy ကို သေချာစေသည်
- **Double Ratchet Algorithm:** တစ်ကြိမ်ပေးပို့ချက်တိုင်း key အသစ် generate ဖြင့် perfect forward secrecy ရရှိသည်

Server သည် encrypt ပြုလုပ်ထားသော ciphertext ကိုသာ သိမ်းဆည်းပြီး plaintext ကို မည်သောအချိန်တွင်မှ မမြင်ရပေ။

---

## ၃၆.၇ Presence System

### Online/Offline Detection

**Heartbeat Mechanism:** Client သည် ၃၀ second တစ်ကြိမ် server သို့ heartbeat ပေးပို့သည်-

```
Client ──▶ WebSocket Server ──▶ Presence Service
               Heartbeat             Redis.setex(
               every 30s             "presence:{user_id}",
                                     "online", TTL=60)
```

Heartbeat ရပ်ဆိုင်းပြီး TTL expire ဖြစ်ပါက User ကို "offline" အဖြစ် သတ်မှတ်သည်။

### Typing Indicators

**Typing event** ကို publish/subscribe pattern ဖြင့် handle ပြုလုပ်သည်-

```
User types ─▶ WS Server ─▶ Redis Pub/Sub
                            Channel: "typing:{conversation_id}"
                            Message: {user_id, is_typing: true}
                                │
                    Other participants ◀─ Subscribe to channel
```

Typing indicator ကို database ထဲ မသိမ်းဆည်းဘဲ Redis Pub/Sub မှတဆင့်သာ broadcast ပြုလုပ်သည်။

---

## ၃၆.၈ Media Upload & Delivery

### Chunked Upload Pipeline

ကြီးမားသော file များကို **Chunked Upload** ဖြင့် upload ပြုလုပ်သည်-

```
Step 1: Client ─▶ Media Service ─▶ Request Upload URL
Step 2: Media Service ─▶ S3/Object Storage ─▶ Generate Presigned URL
Step 3: Client ─▶ S3 (Direct Upload in Chunks) ─▶ Upload Complete
Step 4: Client ─▶ Media Service ─▶ Confirm Upload (media_id)
Step 5: Media Service ─▶ Kafka ─▶ Thumbnail/Processing Job
Step 6: Worker ─▶ CDN ─▶ Distribute processed media
```

### CDN Delivery

- Original file: S3 ထဲ သိမ်းဆည်းသည် (Cold Storage)
- Processed thumbnails/compressed files: CDN Edge Nodes တွင် cache ပြုလုပ်သည်
- CDN URL: `https://cdn.chat.example.com/media/{media_id}/{size}`

---

## ၃၆.၉ Read Receipts at Scale

Read Receipts (မက်ဆေ့ ဖတ်ပြီးကြောင်း အသိပေးခြင်း) သည် **write-heavy** workload ဖြစ်သည်။ DAU ၅၀၀M ရှိပြင် မက်ဆေ့တိုင်း read receipt generate ဖြစ်ပါက ပြဿနာ ဖြစ်နိုင်သည်။

### Optimizations

**Batching:** Read receipts ကို တစ်ချက်ချင်း မပို့ဘဲ batch ဖြင့် (5 second တစ်ကြိမ်) ပေးပို့သည်-

```
Client-side batching:
  - Collect read receipts for 5 seconds
  - Send: {message_ids: [id1, id2, id3], read_at: timestamp}
```

**Async Processing:** Read receipts ကို Kafka queue ထဲ ထည့်ပြီး background worker မှ process ပြုလုပ်သည်-

```
Read Receipt Event ─▶ Kafka ─▶ Receipt Worker ─▶ Update DB
                                                  │
                                                  ▼
                                         Notify Sender via WS
```

**Storage Strategy:** Cassandra ထဲ သိမ်းဆည်းပြီး message_id ဖြင့် query ပြုလုပ်သည်-
```
Table: message_receipts
- message_id (partition key)
- reader_id (clustering key)
- read_at
```

---

## ၃၆.၁၀ Architecture Diagram နှင့် Data Flow

### Full System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT APPS                             │
│              (iOS / Android / Web Browser)                      │
└────────────────────────┬────────────────────────────────────────┘
                         │ WebSocket + HTTPS
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API GATEWAY / LB                           │
│            (Nginx / AWS ALB / Envoy Proxy)                      │
└──────┬──────────┬──────────┬──────────┬──────────┬─────────────┘
       │          │          │          │          │
       ▼          ▼          ▼          ▼          ▼
  ┌─────────┐ ┌──────┐ ┌────────┐ ┌───────┐ ┌──────────┐
  │WS Server│ │ User │ │Message │ │ Group │ │  Media   │
  │(Session)│ │ Svc  │ │  Svc   │ │  Svc  │ │  Svc     │
  └────┬────┘ └──┬───┘ └───┬────┘ └───┬───┘ └────┬─────┘
       │         │         │          │           │
       ▼         ▼         ▼          ▼           ▼
  ┌─────────────────────────────────────────────────────┐
  │              Redis Cluster                          │
  │  Connection Registry | Presence | Session Cache     │
  └─────────────────────────────────────────────────────┘
       │                   │                    │
       ▼                   ▼                    ▼
  ┌─────────┐        ┌──────────┐         ┌──────────┐
  │ Kafka   │        │Cassandra │         │  S3 /    │
  │ Queue   │        │  (msgs)  │         │Object St.│
  └────┬────┘        └──────────┘         └────┬─────┘
       │                                        │
       ▼                                        ▼
  ┌─────────┐                           ┌──────────────┐
  │Notif.   │                           │    CDN       │
  │Service  │                           │(Media Cache) │
  └─────────┘                           └──────────────┘
  APNs/FCM
```

### Message Send Data Flow (1-to-1)

```
1. Alice opens app → WebSocket connect to WS-Server-3
   Redis: SET connection:alice → {server: "WS-3"}

2. Alice sends "Hello Bob" → WS-Server-3 receives

3. WS-Server-3 → Message Service (via Kafka)
   Message: {id: "uuid-123", from: alice, to: bob, content: encrypted}

4. Message Service:
   a. Deduplicate check (Redis)
   b. Persist to Cassandra
   c. Lookup Bob's connection (Redis → "WS-Server-7")

5. Message Service → WS-Server-7 (via internal gRPC/Redis Pub/Sub)

6. WS-Server-7 → Bob's WebSocket → Bob receives message

7. Bob sends ACK → Message Service updates delivery status

8. Delivery receipt sent back to Alice via WS-Server-3
```

---

## Key Design Decisions နှင့် Trade-offs

| Design Decision | ရွေးချယ်မှု | ရည်ရွယ်ချက် |
|---|---|---|
| **Message Storage** | Cassandra | Write-heavy workload, horizontal scaling |
| **Connection Registry** | Redis | Sub-millisecond lookup for routing |
| **Message Queue** | Kafka | High-throughput, durable, replay capable |
| **Encryption** | Signal Protocol (E2EE) | Server မှ plaintext မမြင်ရ |
| **Media Storage** | S3 + CDN | Cost-effective, global distribution |
| **Presence** | Redis TTL + Heartbeat | Lightweight, scalable |

### ပဓာန Trade-offs

- **Consistency vs Availability:** Message ordering တွင် Cassandra ၏ eventual consistency ကို accept ပြုလုပ်ပြီး Group Chat sequence number ဖြင့် compensate ပြုလုပ်သည်
- **Latency vs Durability:** Kafka ကို async ဖြင့် အသုံးပြုခြင်းသည် durability ကောင်းသော်လည်း real-time latency တစ်ချို့ဆုံးရှုံးနိုင်သည်
- **Storage Cost vs Access Speed:** Cold media files ကို S3 Glacier သို့ archive ပြုလုပ်ခြင်းဖြင့် cost လျှော့ချနိုင်ပြီး CDN ဖြင့် hot content ကို fast serve ပြုလုပ်သည်
