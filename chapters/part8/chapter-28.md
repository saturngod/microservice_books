# အခန်း ၂၈: Cloud Storage စနစ် (Dropbox / Google Drive)

## နိဒါန်း

Cloud Storage system သည် ကမ္ဘာ့ biggest distributed systems များထဲမှ တစ်ခုဖြစ်သည်။ Google Drive သည် 1 billion users ကျော်ကို ဝန်ဆောင်မှုပေးပြီး Dropbox သည် exabyte-scale storage ကို manage ပြုလုပ်သည်။ ဤအခန်းတွင် file upload, sync, deduplication, conflict resolution, နှင့် offline support စသည့် complex engineering challenges များကို microservices perspective မှ ဖြေရှင်းနည်းကို လေ့လာမည်ဖြစ်သည်။

Cloud storage ၏ core value proposition မှာ — user ၏ files များကို မည်သည့် device မှမဆို access ပြုနိုင်ပြီး real-time sync ဖြင့် latest version ကို maintain ပြုလုပ်ပေးခြင်းဖြစ်သည်။ ဤ promise ကို fulfill ပြုလုပ်ရန် distributed system design ၏ hardest problems — consistency, availability, partition tolerance — ကို navigate ရမည်ဖြစ်သည်။

---

## ၂၈.၁ Requirements နှင့် Scale Estimation

### Functional Requirements

- **File Upload**: Large files (up to 50 GB) ကို upload ပြုနိုင်ရမည်
- **File Download**: မည်သည့် device မှမဆို files ကို download ပြုနိုင်ရမည်
- **File Sync**: Multiple devices တွင် automatic sync ဖြစ်ရမည်
- **File Sharing**: Files/folders ကို users အချင်းချင်း share ပြုနိုင်ရမည်
- **Version History**: Previous versions ကို restore ပြုနိုင်ရမည်
- **Conflict Resolution**: Multiple devices မှ same file edit ဖြစ်သောအခါ resolve ပြုရမည်

### Non-Functional Requirements

- **Durability**: 99.999999999% (11 nines) — data loss မဖြစ်ရ
- **Availability**: 99.99% uptime
- **Consistency**: Eventual consistency (sync delay < 1 second under normal conditions)
- **Bandwidth Efficiency**: Delta sync — changed parts only transfer

### Scale Estimation

```
Users:
  - Total users: 1 billion
  - Daily Active Users (DAU): 500 million
  - Concurrent users: 50 million

Storage:
  - Average storage per user: 10 GB
  - Total storage: 1B × 10 GB = 10 Exabytes
  - New data/day: 500M DAU × 100 KB avg upload = 50 TB/day

File Operations:
  - Uploads/day: 500M users × 2 files = 1 billion uploads
  - 1B / 86,400s ≈ 11,574 uploads/second
  - Downloads ≈ 5× uploads = 57,870 downloads/second

Bandwidth:
  - Upload bandwidth: 1B files × 100 KB avg = 100 TB/day = 9.3 Gbps
  - Download bandwidth: 500 TB/day = 46.3 Gbps
  - CDN required for download optimization
```

---

## ၂၈.၂ Domain Decomposition

### ၂၈.၂.၁ Upload Service

File upload lifecycle ကို manage ပြုလုပ်သည်။

**Responsibilities:**
- Chunked upload orchestration
- Upload session management
- Deduplication check (before storing)
- Virus/malware scanning (async)
- Storage backend integration

### ၂၈.၂.၂ Metadata Service

File tree structure, permissions, version history ကို manage ပြုလုပ်သည်။

**Responsibilities:**
- File/folder hierarchy management
- Permission model (owner, editor, viewer)
- Version tracking
- Sharing link management

### ၂၈.၂.၃ Sync Service

Client devices တွင် changes ကို propagate ပြုလုပ်သည်။

**Responsibilities:**
- Change detection (file watcher)
- Sync queue management
- Delta calculation
- Conflict detection

### ၂၈.၂.၄ Notification Service

Real-time sync notifications ကို deliver ပြုလုပ်သည်။

**Responsibilities:**
- WebSocket/SSE connections manage
- Push notifications (mobile)
- Email notifications
- Change events broadcasting

### ၂၈.၂.၅ CDN Service

File download ကို edge nodes မှ serve ပြုလုပ်သည်။

**Responsibilities:**
- Presigned URL generation
- Edge caching strategy
- Bandwidth optimization

---

## ၂၈.၃ Chunked Upload နှင့် Resumable Uploads

Large file uploads ကို single HTTP request ဖြင့် ဆောင်ရွက်ခြင်းသည် network failures, timeouts, memory issues များကြောင့် practical မဟုတ်ပါ။

### Chunk Strategy

```
File: 10 GB video file

Step 1: Split into chunks
  - Chunk size: 8 MB (configurable: 1-256 MB)
  - Total chunks: 10 GB / 8 MB = 1,280 chunks
  - Each chunk: identified by chunk_index + file_hash

Step 2: Initiate Upload Session
  POST /api/v1/upload/init
  {
    "file_name": "vacation.mp4",
    "file_size": 10737418240,
    "file_hash": "sha256:abc...",  // whole file hash
    "total_chunks": 1280,
    "chunk_size": 8388608
  }
  Response: {"upload_id": "session-uuid-123"}

Step 3: Upload each chunk
  PUT /api/v1/upload/{upload_id}/chunk/{chunk_index}
  Body: chunk binary data
  Headers: 
    Content-MD5: {chunk_hash}
    Content-Range: bytes 0-8388607/10737418240

Step 4: Complete Upload
  POST /api/v1/upload/{upload_id}/complete
  {"chunk_hashes": ["hash0", "hash1", ...]}
```

### Resumable Upload

Network failure ဖြစ်သောအခါ —

```
GET /api/v1/upload/{upload_id}/status
Response:
{
  "upload_id": "session-uuid-123",
  "uploaded_chunks": [0, 1, 2, 5, 6],  // chunks received
  "missing_chunks": [3, 4, 7, 8, ...], // chunks to re-upload
  "status": "in_progress"
}
```

Client သည် missing chunks ကိုသာ re-upload ပြုသည်။ ပြီးသော chunks ကို ထပ်မ upload မလိုပါ။

---

## ၂၈.၄ File Deduplication — Content-Addressable Storage (CAS)

CAS သည် file content ၏ cryptographic hash ကို storage key အဖြစ် သုံးသော storage model ဖြစ်သည်။ Same content = same hash = same storage location ဖြောင့်မျှ storage ကို 30-40% save ပြုနိုင်သည်။

### Chunk-Level Deduplication

```
File A: [Chunk1, Chunk2, Chunk3, Chunk4]
File B: [Chunk1, Chunk2, Chunk5, Chunk6]  // Chunk1, Chunk2 shared

Storage:
  Chunk1: sha256:aaa → stored once
  Chunk2: sha256:bbb → stored once
  Chunk3: sha256:ccc → stored once
  Chunk4: sha256:ddd → stored once
  Chunk5: sha256:eee → stored once
  Chunk6: sha256:fff → stored once

File A manifest: [aaa, bbb, ccc, ddd]
File B manifest: [aaa, bbb, eee, fff]  ← Chunk1, Chunk2 reuse

Savings: 2 chunks not duplicated = 16 MB saved
```

### Upload Flow with Deduplication

```
1. Client computes chunk hash locally
2. POST /api/v1/chunks/check {hashes: ["aaa", "bbb", ...]}
3. Server returns: {existing: ["aaa", "bbb"], missing: ["ccc", ...]}
4. Client uploads ONLY missing chunks
5. Server assembles file manifest from existing + new chunks
```

**Enterprise example:** 1,000 employees ကို same software (1 GB) distribute ပြုလုပ်သောအခါ storage: 1 GB (1 copy) + 1,000 manifests (< 1 MB) = 99.9% saving

---

## ၂၈.၅ Delta Sync — Changed Blocks Only Transfer

File တစ်ခု update ဖြစ်သောအခါ entire file ကို re-upload မလုပ်ဘဲ changed portions ကိုသာ transfer ပြုသည်။

### Rolling Hash Algorithm (rsync-style)

```
Original File (v1):
  [Block_A][Block_B][Block_C][Block_D]

Modified File (v2):
  [Block_A][Block_B_modified][Block_C][Block_E_new]

Delta Calculation:
  - Block_A: hash unchanged → skip
  - Block_B: hash changed → upload Block_B_modified
  - Block_C: hash unchanged → skip
  - Block_D: removed → record deletion
  - Block_E: new → upload Block_E_new

Delta size: 2 blocks vs 4 blocks = 50% bandwidth saving
```

### Variable-Size Chunking (CDC - Content-Defined Chunking)

Fixed-size chunking တွင် file ၏ beginning တွင် byte တစ်ခု insert ပြုသောအခါ all subsequent chunks shift ဖြစ်သည်။ CDC သည် ဤပြဿနာကို ဖြေရှင်းသည်။

```
CDC uses a rolling hash to find chunk boundaries:
  - Chunk boundary: When hash(sliding_window) & mask == 0
  - Average chunk size: 4-8 MB
  - Content-dependent → insertions don't affect downstream chunks
```

---

## ၂၈.၆ Conflict Resolution

Multiple devices မှ same file ကို simultaneously edit ပြုသောအခါ conflict ဖြစ်သည်။

### Last Write Wins (LWW)

```
Device A: Edit file at 10:00:00
Device B: Edit file at 10:00:01  ← wins

Server accepts B's version.
A's changes silently discarded.
```

**Problem:** Data loss ဖြစ်သည်။ Design ညံ့သော်လည်း simple applications တွင် acceptable ဖြစ်သည်။

### Branching (Google Drive approach)

```
Device A edits: Creates version "v2_deviceA"
Device B edits: Creates version "v2_deviceB"

Both versions preserved:
  file.txt (Device A's version)
  file (Device B's version - 2026-04-06).txt

User manually merges or picks one.
```

**Problem:** User confusion၊ duplicate files များ။ သို့သော် data loss မဖြစ်ပါ။

### Operational Transform / CRDT (Advanced)

Google Docs ကဲ့သို့ real-time collaborative editing တွင် CRDT (Conflict-free Replicated Data Type) ကို သုံးသည်။ Operations ကို commutative အဖြစ် model ပြုလုပ်ခြင်းဖြင့် automatic merge ဖြစ်သည်။

**Production recommendation:** Files တွင် Branching ကို သုံးပြီး documents တွင် CRDT ကို သုံးသည်။

---

## ၂၈.၇ Offline Support နှင့် Local Cache

### Local Sync Client Architecture

```
┌─────────────────────────────────────────┐
│           Sync Client (Desktop/Mobile)   │
│                                          │
│  ┌──────────────┐  ┌──────────────────┐ │
│  │ File Watcher │  │  Sync Queue      │ │
│  │ (inotify/    │  │  (SQLite local   │ │
│  │  FSEvents)   │  │   DB)            │ │
│  └──────┬───────┘  └────────┬─────────┘ │
│         │                   │           │
│  ┌──────▼───────────────────▼─────────┐ │
│  │          Sync Engine               │ │
│  │  - Conflict detection              │ │
│  │  - Delta calculation               │ │
│  │  - Upload/Download queue           │ │
│  └──────────────────┬─────────────────┘ │
│                     │                   │
│  ┌──────────────────▼─────────────────┐ │
│  │        Local Cache (LRU)           │ │
│  │   Recently used files (hot)        │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Offline Queue

```
State Machine:
  PENDING_SYNC → UPLOADING → SYNCED
  PENDING_SYNC → CONFLICT_DETECTED → USER_RESOLVE → SYNCED

Local SQLite Schema:
  CREATE TABLE pending_changes (
    change_id    TEXT PRIMARY KEY,
    file_path    TEXT,
    change_type  TEXT, -- 'create', 'update', 'delete', 'rename'
    local_hash   TEXT,
    server_hash  TEXT, -- null if not yet synced
    status       TEXT, -- 'pending', 'uploading', 'conflict'
    retry_count  INTEGER DEFAULT 0,
    created_at   DATETIME
  );
```

---

## ၂၈.၈ Storage Backend — S3 + Block Storage

### Multi-Tier Storage Strategy

```
Hot Storage (S3 Standard):
  - Recently accessed files (< 30 days)
  - Latency: < 5ms
  - Cost: $0.023/GB/month

Warm Storage (S3 Infrequent Access):
  - Files accessed rarely (30-90 days)
  - Latency: < 5ms
  - Cost: $0.0125/GB/month (45% cheaper)

Cold Storage (S3 Glacier):
  - Archive files (> 90 days)
  - Retrieval time: minutes to hours
  - Cost: $0.004/GB/month (83% cheaper)

Transition Rules (S3 Lifecycle Policy):
  - 30 days → Move to IA
  - 90 days → Move to Glacier
  - 7 years → Delete (legal retention)
```

### Geographic Distribution

```
Data Residency (GDPR/regional compliance):
  - EU users → EU regions (Frankfurt, Ireland)
  - US users → US regions (Virginia, Oregon)
  - APAC users → Singapore, Tokyo

Replication:
  - S3 Cross-Region Replication (CRR)
  - Minimum 3 copies in different availability zones
  - 11 nines durability guarantee
```

---

## ၂၈.၉ Metadata Service Design

### File Tree Model

```sql
CREATE TABLE files (
    file_id      UUID PRIMARY KEY,
    parent_id    UUID REFERENCES files(file_id),
    owner_id     UUID,
    name         TEXT NOT NULL,
    type         TEXT, -- 'file', 'folder'
    size         BIGINT,
    mime_type    TEXT,
    content_hash TEXT, -- SHA-256 of content
    created_at   TIMESTAMP,
    updated_at   TIMESTAMP,
    is_deleted   BOOLEAN DEFAULT FALSE,
    version      INTEGER DEFAULT 1
);

CREATE TABLE file_permissions (
    file_id      UUID REFERENCES files(file_id),
    user_id      UUID,
    permission   TEXT, -- 'owner', 'editor', 'viewer', 'commenter'
    granted_by   UUID,
    granted_at   TIMESTAMP,
    expires_at   TIMESTAMP,
    PRIMARY KEY (file_id, user_id)
);

CREATE TABLE file_versions (
    file_id      UUID REFERENCES files(file_id),
    version      INTEGER,
    content_hash TEXT,
    size         BIGINT,
    created_by   UUID,
    created_at   TIMESTAMP,
    chunk_manifest JSONB, -- ordered list of chunk hashes
    PRIMARY KEY (file_id, version)
);
```

### Permission Inheritance

```
Folder: /Projects (editor permission for Team A)
  └─ Subfolder: /Projects/App (inherited: Team A = editor)
     └─ File: design.figma (inherited + explicit: User B = viewer)
```

---

## ၂၈.၁၀ Architecture Diagram နှင့် Data Flow

### Full System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│         Web Browser / Desktop App / Mobile App                  │
└──────────┬──────────────────────────────┬───────────────────────┘
           │                              │
           │ Upload/API                   │ WebSocket (Sync)
           │                              │
┌──────────▼──────────────────────────────▼───────────────────────┐
│                    API Gateway / Load Balancer                    │
│              (Auth, Rate Limit, SSL Termination)                  │
└────┬──────────┬──────────────┬───────────────────────────────────┘
     │          │              │
┌────▼──┐  ┌───▼───┐   ┌──────▼────────┐   ┌────────────────────┐
│Upload │  │Metadata│   │ Sync Service  │   │Notification Service│
│Service│  │Service │   │               │   │  (WebSocket/Push)  │
└────┬──┘  └───┬────┘   └──────┬────────┘   └──────────┬─────────┘
     │          │              │                        │
┌────▼──────────▼──────────────▼────────────────────────▼─────────┐
│                      Message Bus (Kafka)                          │
│         Topics: file-events, sync-events, notifications           │
└──────────────────────────────────────────────────────────────────┘
           │                              │
┌──────────▼──────────┐         ┌────────▼────────────────────────┐
│   Storage Backend   │         │        Metadata Database         │
│  ┌────────────────┐ │         │   PostgreSQL (with Citus)        │
│  │   S3/Object    │ │         │   or CockroachDB                 │
│  │   Storage      │ │         └─────────────────────────────────┘
│  │  (Multi-tier)  │ │
│  └────────────────┘ │
│  ┌────────────────┐ │
│  │  CDN (CloudFront│ │
│  │  /Fastly)      │ │
│  └────────────────┘ │
└─────────────────────┘
```

### File Upload Data Flow

```
1. Client: Split file into chunks, compute hashes locally
2. Client → Upload Service: Check which chunks exist (dedup check)
3. Upload Service → Metadata Service: Check file version, conflicts
4. Client → Upload Service: Upload only missing chunks
5. Upload Service → S3: Store chunks (Content-Addressed key)
6. Upload Service → Metadata Service: Update file manifest, version
7. Metadata Service → Kafka: Publish "file-updated" event
8. Kafka → Sync Service: Consume event
9. Sync Service → Notification Service: Notify other devices
10. Other devices: Download delta (changed chunks only)
```

---

## Key Design Decisions နှင့် Trade-offs

| Challenge | Solution | Trade-off |
|---|---|---|
| Large file upload | Chunked + Resumable | Complexity vs Reliability |
| Storage cost | CAS Deduplication | CPU cost vs Storage savings |
| Bandwidth | Delta Sync (CDC) | Complexity vs Network efficiency |
| Conflict | Branching | User experience vs Data safety |
| Latency | CDN + Tiered storage | Cost vs Performance |
| Consistency | Eventual consistency | Complexity vs Availability |

### Cold Path vs Hot Path

```
Hot Path (Frequently accessed):
  S3 Standard + CDN → < 50ms latency
  Cache presigned URLs for 1 hour

Cold Path (Archive access):
  S3 Glacier → Trigger restore job
  Restore time: 3-5 minutes (expedited)
  Notify user when ready
```

## အနှစ်ချုပ်

Cloud storage system ၏ engineering complexity မှာ — single file operation ကို distributed system ၏ multiple layers (upload chunking, dedup, metadata, sync, notification, CDN) ဖြတ်သန်း ဆောင်ရွက်ရခြင်းဖြစ်သည်။ CAS-based deduplication ဖြင့် storage cost ကို significantly reduce ပြုနိုင်ပြီး delta sync ဖြင့် bandwidth consumption ကို minimize ပြုနိုင်သည်။ 11-nines durability ကို achieve ပြုရန် multi-AZ replication နှင့် immutable chunk storage ကို combine ပြုလုပ်ရသည်။ Conflict resolution ၌ user experience နှင့် data safety ကြားတွင် balance ကို ရှာဖွေရသည်မှာ ဤ system ၏ hardest design problem ဖြစ်သည်။
