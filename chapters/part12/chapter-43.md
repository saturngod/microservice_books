# အခန်း ၄၃ — Photo Sharing Platform (Instagram ကဲ့သို့) ဒီဇိုင်း

## နိဒါန်း

Photo sharing platform သည် media-heavy, fan-out intensive, real-time personalization လိုအပ်သော complex distributed system ဖြစ်သည်။ Instagram သည် 2 billion monthly active users ရှိပြီး daily 100 million photos/videos upload ဖြစ်သည်။ ဤအခန်းတွင် photo upload pipeline, feed generation (fanout), stories architecture, explore page personalization, reels pipeline, နှင့် full system architecture တို့ကို microservices perspective မှ design ပြုလုပ်သွားမည်ဖြစ်သည်။

---

## ₄₃.₁ Requirements

### Functional Requirements

- **Photo/Video Upload:** High-quality photos + short videos upload
- **Feed:** Following list posts ကို personalized order ဖြင့် ပြသ
- **Stories:** 24-hour ephemeral content (photos/short videos)
- **Reels:** Short-form video (15-90 seconds)
- **Explore Page:** Interest-based content discovery
- **Notifications:** Likes, comments, follows, mentions
- **Direct Messages:** Private messaging (text, photo, video)
- **Comments:** Threaded comments on posts
- **Search:** People, hashtags, places search

### Non-Functional Requirements

| Metric | Target |
|---|---|
| Daily Active Users (DAU) | 100 million |
| Monthly Active Users (MAU) | 500 million |
| Photo Uploads/Day | 100 million |
| Feed Reads/Day | 5 billion |
| Stories Views/Day | 1 billion |
| Photo Upload Latency | < 5 seconds |
| Feed Load Latency | < 1 second |
| Storage (new data/day) | ~100 TB |
| System Availability | 99.99% |

### Scale Estimation

```
Upload throughput:
  100M photos/day / 86,400 = ~1,157 uploads/sec
  Peak (3x): ~3,500 uploads/sec
  Avg photo size: 2 MB
  Storage: 100M * 2MB = 200 TB/day (originals)
  With thumbnails (4 sizes): ~300 TB/day

Feed reads:
  5B reads/day / 86,400 = ~57,870 reads/sec
  Peak: 3x = ~174,000 reads/sec
  Read:Write ratio = ~50:1

Fan-out load:
  Avg user has 500 followers
  100M posts * 500 followers = 50B fan-out writes/day
  Celebrity (5M+ followers): pull-based (not fan-out)

Stories:
  1B views/day / 86,400 = ~11,574 views/sec
  Stories expire after 24h: TTL-based cleanup
```

---

## ₄₃.₂ Domain Decomposition

```
Instagram Domain Services:
┌──────────────────────────────────────────────────────────────┐
│                      API Gateway (BFF)                       │
│              Mobile BFF │ Web BFF                             │
└──┬────┬────┬────┬────┬────┬────┬────┬────────────────────────┘
   │    │    │    │    │    │    │    │
┌──▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──────────┐
│Media││Feed││Disc││Story││Noti││DM  ││Cmnt││  User    │
│ Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││  Svc     │
└─────┘└────┘└────┘└────┘└────┘└────┘└────┘└──────────┘
                                        │
                              ┌─────────▼──────────┐
                              │   Search Service   │
                              │   (Elasticsearch)  │
                              └────────────────────┘
```

| Service | Responsibility | Database |
|---|---|---|
| **Media Service** | Upload, processing, storage | S3 + PostgreSQL (metadata) |
| **Feed Service** | Feed generation, fanout, timeline | Redis (cache) + Cassandra |
| **Discover Service** | Explore page, trending, personalization | Elasticsearch + ML model |
| **Story Service** | Stories CRUD, 24h TTL, view tracking | Redis (TTL) + Cassandra |
| **Notification Service** | Push, in-app notifications | Redis queue + FCM/APNs |
| **DM Service** | Direct messaging, group chats | Cassandra + Redis (presence) |
| **Comment Service** | Comments, replies, threading | Cassandra |
| **User Service** | Profiles, follows, authentication | PostgreSQL + Redis (social graph) |
| **Search Service** | People, hashtags, places search | Elasticsearch |

---

## ₄₃.₃ Photo Upload Pipeline

```
Photo Upload Pipeline:
┌────────┐   Chunked    ┌───────────┐   Store    ┌──────────┐
│ Client │ ──Upload────> │  Upload   │ ────────> │   S3     │
│ (App)  │              │  Service  │           │(Original)│
└────────┘              └─────┬─────┘           └──────────┘
                              │
                         ┌────▼─────┐
                         │  Kafka   │
                         │ (event)  │
                         └────┬─────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
        ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
        │ Thumbnail │  │   EXIF    │  │  Content  │
        │ Generator │  │ Extractor │  │ Moderator │
        │ (4 sizes) │  │(GPS,Date) │  │ (ML/NSFW) │
        └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
              │               │               │
        ┌─────▼─────┐        │          Pass/Fail
        │   CDN     │        │               │
        │ (edge)    │        │               │
        └───────────┘   ┌────▼───────────────▼──┐
                        │   Media Metadata      │
                        │   (PostgreSQL)        │
                        │   status: published   │
                        └───────────┬───────────┘
                                    │
                              ┌─────▼─────┐
                              │  Fan-out  │
                              │  Service  │
                              └───────────┘
```

```python
# Photo upload service — chunked upload + async processing
import boto3
from PIL import Image
import io

class MediaUploadService:
    """Photo upload → S3 → async processing pipeline"""

    def __init__(self, s3_client, kafka_producer):
        self.s3 = s3_client
        self.kafka = kafka_producer
        self.bucket = "instagram-media"

    async def upload_photo(self, user_id, photo_data, caption, tags):
        """Photo upload handle — S3 save + async processing trigger"""
        media_id = str(uuid.uuid4())

        # Step 1: Original photo → S3
        s3_key = f"originals/{user_id}/{media_id}.jpg"
        self.s3.put_object(
            Bucket=self.bucket,
            Key=s3_key,
            Body=photo_data,
            ContentType="image/jpeg",
            ServerSideEncryption="AES256",
        )

        # Step 2: Metadata save (status = PROCESSING)
        media = Media(
            id=media_id, user_id=user_id,
            s3_key=s3_key, caption=caption, tags=tags,
            status="PROCESSING",
            created_at=datetime.utcnow(),
        )
        await self.media_repo.save(media)

        # Step 3: Async processing trigger
        await self.kafka.send("media.uploaded", {
            "media_id": media_id,
            "user_id": user_id,
            "s3_key": s3_key,
            "type": "photo",
        })

        return {"media_id": media_id, "status": "PROCESSING"}

    async def process_photo(self, event):
        """Async photo processing — thumbnails + moderation + EXIF"""
        media_id = event["media_id"]

        # Download original from S3
        original = self.s3.get_object(Bucket=self.bucket, Key=event["s3_key"])
        image_data = original["Body"].read()
        img = Image.open(io.BytesIO(image_data))

        # Generate thumbnails (4 sizes)
        thumbnail_sizes = {
            "small": (150, 150),     # Profile grid
            "medium": (640, 640),    # Feed (mobile)
            "large": (1080, 1080),   # Feed (tablet/web)
            "xlarge": (1920, 1920),  # Full screen
        }

        thumbnail_urls = {}
        for size_name, dimensions in thumbnail_sizes.items():
            thumbnail = img.copy()
            thumbnail.thumbnail(dimensions, Image.LANCZOS)

            thumb_key = f"thumbnails/{event['user_id']}/{media_id}/{size_name}.jpg"
            buffer = io.BytesIO()
            thumbnail.save(buffer, format="JPEG", quality=85)
            buffer.seek(0)

            self.s3.put_object(
                Bucket=self.bucket, Key=thumb_key,
                Body=buffer.getvalue(), ContentType="image/jpeg",
                CacheControl="public, max-age=31536000",
            )
            thumbnail_urls[size_name] = f"https://cdn.instagram.com/{thumb_key}"

        # Content moderation (ML)
        moderation = await self.content_moderator.check(image_data)
        if not moderation.safe:
            await self.media_repo.update_status(media_id, "REJECTED")
            return

        # EXIF extraction
        exif_data = self._extract_exif(img)

        # Update metadata → PUBLISHED
        await self.media_repo.update(media_id, {
            "thumbnail_urls": thumbnail_urls,
            "exif_data": exif_data,
            "status": "PUBLISHED",
        })

        # Trigger fan-out
        await self.kafka.send("media.published", {
            "media_id": media_id,
            "user_id": event["user_id"],
            "thumbnail_urls": thumbnail_urls,
        })
```

---

## ₄₃.₄ Feed Generation (Fanout)

### Push-Pull Hybrid Strategy

```
Fanout Strategy:
┌──────────────────────────────────────────────────────────────┐
│  Push Fanout (Regular Users — < 5M followers)                │
│                                                              │
│  User A posts photo                                          │
│  ├──> Follower 1's feed cache (Redis ZADD)                   │
│  ├──> Follower 2's feed cache                                │
│  └──> ... (all followers)                                    │
│                                                              │
│  Feed read = Redis ZRANGE (fast, O(log N + K))               │
├──────────────────────────────────────────────────────────────┤
│  Pull Fanout (Celebrities — >= 5M followers)                 │
│                                                              │
│  Celebrity posts → save to own timeline only                 │
│  Follower reads feed:                                        │
│  ├──> Read push cache + Query celebrities' timelines         │
│  └──> Merge + sort → return                                  │
└──────────────────────────────────────────────────────────────┘
```

```python
# Feed service — push-pull hybrid
class FeedService:
    """Feed generation — push-pull hybrid fanout"""

    CELEBRITY_THRESHOLD = 5_000_000

    def __init__(self, redis_client, social_graph, media_repo):
        self.redis = redis_client
        self.social_graph = social_graph
        self.media_repo = media_repo

    async def fan_out_post(self, user_id, media_id, published_at):
        """Post published → followers' feeds ထဲ fan-out"""
        follower_count = await self.social_graph.get_follower_count(user_id)

        if follower_count >= self.CELEBRITY_THRESHOLD:
            # Celebrity — own timeline save only
            self.redis.zadd(
                f"user_timeline:{user_id}",
                {media_id: published_at.timestamp()}
            )
            return

        # Regular user — push to all followers
        followers = await self.social_graph.get_followers(user_id)

        # Batch 1000 followers per pipeline
        batch_size = 1000
        for i in range(0, len(followers), batch_size):
            batch = followers[i:i + batch_size]
            pipe = self.redis.pipeline()
            for follower_id in batch:
                pipe.zadd(f"feed:{follower_id}", {media_id: published_at.timestamp()})
                pipe.zremrangebyrank(f"feed:{follower_id}", 0, -501)  # Keep 500
            pipe.execute()

    async def get_feed(self, user_id, page=1, size=20):
        """User feed — push entries + celebrity pull merge"""
        offset = (page - 1) * size

        # Push feed entries (Redis)
        push_ids = self.redis.zrevrange(f"feed:{user_id}", offset, offset + size - 1)

        # Pull celebrity posts
        celebrities = await self.social_graph.get_followed_celebrities(
            user_id, threshold=self.CELEBRITY_THRESHOLD
        )

        celebrity_posts = []
        for celeb_id in celebrities:
            recent = self.redis.zrevrangebyscore(
                f"user_timeline:{celeb_id}",
                "+inf", time.time() - 86400,
                start=0, num=5, withscores=True,
            )
            celebrity_posts.extend(recent)

        # Merge + sort by timestamp
        all_posts = []
        for pid in push_ids:
            score = self.redis.zscore(f"feed:{user_id}", pid)
            all_posts.append((pid, score))
        all_posts.extend(celebrity_posts)
        all_posts.sort(key=lambda x: x[1], reverse=True)

        # ML ranking (optional)
        ranked = await self._rank_feed(user_id, all_posts[:size * 2])

        # Fetch media details
        return await self.media_repo.get_batch([p for p, _ in ranked[:size]])

    async def _rank_feed(self, user_id, candidates):
        """ML-based feed ranking — engagement prediction"""
        scored = []
        for post_id, ts in candidates:
            meta = await self.media_repo.get_metadata(post_id)
            score = (
                self._recency_score(ts) * 0.3 +
                self._affinity_score(user_id, meta.author_id) * 0.3 +
                self._engagement_score(meta) * 0.25 +
                self._content_type_pref(user_id, meta.type) * 0.15
            )
            scored.append((post_id, score))
        scored.sort(key=lambda x: x[1], reverse=True)
        return scored
```

---

## ₄₃.₅ Stories — 24-Hour Expiry

```
Stories Architecture:
  Story Ring Buffer (Redis):
  ┌────┐ ┌────┐ ┌────┐ ┌────┐
  │ S1 │→│ S2 │→│ S3 │→│ S4 │   (ordered by time)
  │22h │ │18h │ │12h │ │ 2h │   (TTL remaining)
  └────┘ └────┘ └────┘ └────┘
  S1 expires in 2 hours (auto-cleanup via Redis TTL)
```

```python
# Stories — TTL-based 24h expiry + view tracking
class StoryService:
    """Instagram Stories — 24h ephemeral content"""

    STORY_TTL = 86400  # 24 hours

    async def create_story(self, user_id, media_data, media_type="photo"):
        """Story create with 24h TTL"""
        story_id = str(uuid.uuid4())
        created_at = time.time()

        # Media → S3
        s3_key = f"stories/{user_id}/{story_id}.{media_type}"
        self.s3.put_object(
            Bucket="instagram-stories", Key=s3_key,
            Body=media_data,
            Expires=datetime.utcnow() + timedelta(hours=25),
        )
        media_url = f"https://cdn.instagram.com/{s3_key}"

        # Redis save with TTL
        story_data = json.dumps({
            "story_id": story_id, "user_id": user_id,
            "media_url": media_url, "media_type": media_type,
            "created_at": created_at, "view_count": 0,
        })

        pipe = self.redis.pipeline()
        pipe.setex(f"story:{story_id}", self.STORY_TTL, story_data)
        pipe.zadd(f"user_stories:{user_id}", {story_id: created_at})
        pipe.expire(f"user_stories:{user_id}", self.STORY_TTL)
        pipe.execute()

        return {"story_id": story_id, "expires_at": created_at + self.STORY_TTL}

    async def get_stories_feed(self, user_id):
        """Following list stories — ring format with seen/unseen"""
        following = await self.social_graph.get_following(user_id)

        stories_feed = []
        pipe = self.redis.pipeline()
        for fid in following:
            pipe.zrangebyscore(f"user_stories:{fid}", time.time() - self.STORY_TTL, "+inf")
        results = pipe.execute()

        for fid, story_ids in zip(following, results):
            if story_ids:
                seen = self.redis.smembers(f"story_seen:{user_id}:{fid}")
                has_unseen = any(sid not in seen for sid in story_ids)
                stories_feed.append({
                    "user_id": fid,
                    "story_count": len(story_ids),
                    "has_unseen": has_unseen,
                })

        # Unseen first, then by recency
        stories_feed.sort(key=lambda x: (not x["has_unseen"]))
        return stories_feed

    async def view_story(self, viewer_id, story_id):
        """View record + counter increment"""
        story_data = self.redis.get(f"story:{story_id}")
        if not story_data:
            raise StoryExpiredError("Story has expired")

        story = json.loads(story_data)
        # Atomic view count increment
        story["view_count"] = int(story["view_count"]) + 1
        remaining_ttl = self.redis.ttl(f"story:{story_id}")
        self.redis.setex(f"story:{story_id}", remaining_ttl, json.dumps(story))

        # Mark as seen
        self.redis.sadd(f"story_seen:{viewer_id}:{story['user_id']}", story_id)
        self.redis.expire(f"story_seen:{viewer_id}:{story['user_id']}", self.STORY_TTL)

        return story
```

---

## ₄₃.₆ Explore Page — Real-Time Personalization

```python
# Explore page — personalized content discovery
class ExploreService:
    """Explore page — ML-ranked personalized discovery"""

    async def get_explore_feed(self, user_id, page=1, size=30):
        """Personalized explore — interest-based"""

        interests = await self._get_user_interests(user_id)

        # Candidate generation — multiple sources
        candidates = []

        # Source 1: Interest-based (Elasticsearch)
        interest_posts = self.es.search(index="posts", body={
            "query": {
                "bool": {
                    "should": [
                        {"terms": {"tags": interests["hashtags"][:20]}},
                        {"terms": {"category": interests["categories"][:5]}},
                    ],
                    "must_not": [
                        {"terms": {"_id": self._get_recent_seen(user_id)}},
                        {"term": {"user_id": user_id}},
                    ],
                    "filter": [
                        {"range": {"created_at": {"gte": "now-7d"}}},
                        {"range": {"engagement_score": {"gte": 0.1}}},
                    ]
                }
            },
            "size": 100,
        })
        candidates.extend([h["_source"] for h in interest_posts["hits"]["hits"]])

        # Source 2: Trending in location
        trending = await self._get_trending_posts(interests.get("location"), 30)
        candidates.extend(trending)

        # Source 3: Collaborative filtering
        cf_posts = await self._collaborative_filter(user_id, 30)
        candidates.extend(cf_posts)

        # ML ranking
        scored = self._rank_candidates(user_id, candidates)

        # Diversity — max 2 posts per author
        diversified = self._diversify(scored, max_per_author=2)

        return diversified[(page-1)*size : page*size]

    def _rank_candidates(self, user_id, candidates):
        """ML engagement prediction"""
        features = [{
            "user_affinity": self._affinity(user_id, p["user_id"]),
            "post_age_hours": (time.time() - p["created_at"]) / 3600,
            "like_count": p.get("like_count", 0),
            "comment_count": p.get("comment_count", 0),
        } for p in candidates]

        scores = self.ml_model.predict(features)
        for post, score in zip(candidates, scores):
            post["ranking_score"] = float(score)

        return sorted(candidates, key=lambda x: x["ranking_score"], reverse=True)

    def _diversify(self, ranked, max_per_author=2):
        """Feed diversity — author limit"""
        result, counts = [], {}
        for post in ranked:
            author = post["user_id"]
            if counts.get(author, 0) < max_per_author:
                result.append(post)
                counts[author] = counts.get(author, 0) + 1
        return result
```

---

## ₄₃.₇ Reels — Short-Form Video Pipeline

```
Reels Pipeline:
Client ─Chunked Upload─> Upload Svc ─> S3 (Raw) ─> Kafka
                                                      │
                                               ┌──────▼──────┐
                                               │  Transcoding │
                                               │   Pipeline   │
                                               └──────┬──────┘
                                  ┌────────────────────┼────────────┐
                            ┌─────▼─────┐  ┌──────▼──────┐  ┌─────▼──────┐
                            │  1080p    │  │   720p      │  │   480p     │
                            │  HLS     │  │  HLS        │  │  HLS       │
                            └─────┬─────┘  └──────┬──────┘  └─────┬──────┘
                                  └────────────────┼──────────────┘
                                            ┌──────▼──────┐
                                            │  CDN (ABR)  │
                                            └─────────────┘
```

```python
# Reels video processing
class ReelsService:
    """Short-form video (15-90s) upload + transcoding"""

    MAX_DURATION = 90  # seconds

    async def upload_reel(self, user_id, video_data, caption, audio_id=None):
        """Reel upload + async transcoding"""
        reel_id = str(uuid.uuid4())

        duration = self._get_video_duration(video_data)
        if duration > self.MAX_DURATION:
            raise ValidationError(f"Max {self.MAX_DURATION}s, got {duration}s")

        # Raw video → S3
        raw_key = f"reels/raw/{user_id}/{reel_id}.mp4"
        self.s3.put_object(Bucket="instagram-reels", Key=raw_key, Body=video_data)

        # Metadata save
        reel = Reel(
            id=reel_id, user_id=user_id, raw_s3_key=raw_key,
            duration=duration, caption=caption, audio_id=audio_id,
            status="PROCESSING",
        )
        await self.reel_repo.save(reel)

        # Trigger transcoding
        await self.kafka.send("reel.uploaded", {
            "reel_id": reel_id, "user_id": user_id,
            "raw_s3_key": raw_key, "duration": duration,
        })

        return {"reel_id": reel_id, "status": "PROCESSING"}

    async def transcode_reel(self, event):
        """Multi-resolution HLS transcoding"""
        reel_id = event["reel_id"]
        resolutions = {
            "1080p": {"scale": "1920:1080", "bitrate": "4500k"},
            "720p":  {"scale": "1280:720",  "bitrate": "2500k"},
            "480p":  {"scale": "854:480",   "bitrate": "1200k"},
            "360p":  {"scale": "640:360",   "bitrate": "600k"},
        }

        for res, config in resolutions.items():
            await self._transcode_to_hls(
                input_key=event["raw_s3_key"],
                output_prefix=f"reels/hls/{reel_id}/{res}/",
                scale=config["scale"],
                bitrate=config["bitrate"],
            )

        # Content moderation
        moderation = await self.content_moderator.check_video(event["raw_s3_key"])
        if not moderation.safe:
            await self.reel_repo.update_status(reel_id, "REJECTED")
            return

        # Update → PUBLISHED
        await self.reel_repo.update(reel_id, {
            "hls_url": f"https://cdn.instagram.com/reels/hls/{reel_id}/master.m3u8",
            "thumbnail_url": await self._generate_thumbnail(event["raw_s3_key"]),
            "status": "PUBLISHED",
        })

        # Fan-out
        await self.kafka.send("reel.published", {
            "reel_id": reel_id, "user_id": event["user_id"],
        })
```

---

## ₄₃.₈ Architecture Diagram

```
Instagram-like Platform — Full Architecture:
┌─────────────────────────────────────────────────────────────────────────┐
│                         CDN (CloudFront)                               │
│              Photos │ Thumbnails │ Videos │ HLS Segments               │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────────┐
│                       API Gateway (BFF)                                 │
│              Mobile BFF │ Web BFF │ Auth │ Rate Limit                   │
└──┬────┬────┬────┬────┬────┬────┬────┬──────────────────────────────────┘
   │    │    │    │    │    │    │    │
┌──▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──┐┌▼──────┐
│Media││Feed││Disc││Story││Noti││DM  ││Cmnt││User  │
│ Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││Svc ││Svc   │
└──┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└──┬───┘
   │     │     │     │     │     │     │      │
┌──▼──┐┌─▼───┐┌▼───┐┌▼───┐┌▼───┐┌▼────┐┌▼───┐┌─▼───┐
│S3   ││Redis ││ES  ││Redis││Redis││Cass ││Cass││PgSQL│
│+PgSQ││+Cass ││+ML ││TTL ││Queue││andra││ndra││+Rds │
└─────┘└──────┘└────┘└────┘└─────┘└─────┘└────┘└─────┘
                │
         ┌──────▼────────────────────────────┐
         │          Kafka Event Bus           │
         │  media.* │ feed.* │ story.* │ reel.*│
         └──────────┬────────────────────────┘
                    │
          ┌─────────▼──────────┐
          │  Processing Workers│
          │  Thumbnail Gen     │
          │  Video Transcoding │
          │  Content Moderation│
          │  Fan-out Workers   │
          └────────────────────┘

Data Flow:
  Upload: Client → API GW → Media Svc → S3 → Kafka → Workers → CDN
  Feed:   Client → Feed Svc → Redis (push) + Celebrity pull → Merge → Client
  Story:  Client → Story Svc → Redis TTL 24h + S3/CDN → Client
  Reel:   Client → Reels Svc → S3 → FFmpeg HLS → CDN ABR → Client
  Explore: Client → Discover Svc → ES + ML ranking → Client
```

---

## Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Feed** | Push-Pull Hybrid | Balance fanout cost + read latency |
| **Celebrity Threshold** | 5M followers | Avoid massive fan-out |
| **Stories** | Redis + TTL | Auto-expire, fast read/write |
| **Feed Cache** | Redis Sorted Set | O(log N) insert, O(1) range |
| **Media** | S3 + CDN | Infinite scale, global edge |
| **Video** | HLS Adaptive Bitrate | Cross-device streaming |
| **Explore** | ML + Elasticsearch | Personalization + fast retrieval |

---

## အဓိက အချက်များ (Key Takeaways)

1. **Photo upload pipeline** — chunked upload → S3 → async thumbnails (4 sizes) → content moderation → CDN ဖြင့် scalable media handling

2. **Feed generation** — push-pull hybrid fanout: regular users (push to Redis), celebrities (pull at read time) ဖြင့် optimal balance

3. **Stories** — Redis TTL (24h) auto-expire + view counter + seen/unseen tracking ဖြင့် ephemeral content manage

4. **Explore page** — Elasticsearch candidates + ML ranking + diversity injection ဖြင့် personalized discovery

5. **Reels** — upload → multi-resolution transcoding → HLS adaptive bitrate → CDN streaming ဖြင့် video delivery

6. **Scale**: 100M DAU, 100M uploads/day, 5B feed reads/day ကို Redis + Cassandra + S3 + CDN + Kafka ဖြင့် handle

---

> **Part 12 Summary:** Chapter 42 (Twitter) နှင့် Chapter 43 (Instagram) နှစ်ခုလုံး fan-out problem ကို push-pull hybrid ဖြင့် solve ပြုလုပ်သည်။ Twitter = text-centric, Instagram = media-centric (photo/video pipeline)။ Redis = caching backbone, Kafka = async event backbone။
