# အခန်း ၄၂: Social Media Feed ဒီဇိုင်း (Twitter / X ကဲ့သို့)

## နိဒါန်း

ဤအခန်းတွင် Twitter (ယခု X) ကဲ့သို့သော real-time social media platform တစ်ခုကို microservices architecture ဖြင့် ဒီဇိုင်းရေးဆွဲပုံကို အသေးစိတ် လေ့လာပါမည်။ နေ့စဉ် tweet ၅၀၀ သန်း ရေးသားခြင်း၊ sub-second feed delivery ပြုလုပ်ခြင်း၊ trending topics real-time detect လုပ်ခြင်းတို့သည် ဤ system ၏ core engineering challenges ဖြစ်ပါသည်။ အထူးသဖြင့် **fan-out problem** — celebrity user တစ်ဦး tweet ရေးသောအခါ follower သန်းနှင့်ချီ ၏ timeline ကို update လုပ်ရခြင်းသည် system design ၏ အခက်ခဲဆုံး ပြဿနာ ဖြစ်ပါသည်။

## ၄၂.၁ Requirements နှင့် Scale Estimation

### Functional Requirements (လုပ်ဆောင်ချက်ဆိုင်ရာ လိုအပ်ချက်များ)

- **Tweet Posting**: Text (280 chars), images, videos, polls ပါသော tweet များ post ပြုလုပ်နိုင်ရမည်
- **Home Timeline**: Follow ပြုလုပ်ထားသော accounts ၏ tweets ကို feed အဖြစ် ပြသနိုင်ရမည်
- **Follow/Unfollow**: User-to-user subscription relationship
- **Retweet/Like/Reply**: Engagement actions
- **Full-Text Search**: Tweet content နှင့် hashtag search
- **Trending Topics**: Real-time trending hashtags/topics detection
- **Notifications**: Likes, retweets, mentions, new follower alerts
- **Direct Messages**: Private messaging (end-to-end encryption optional)

### Non-Functional Requirements

- **DAU**: ၂၅၀ သန်း Daily Active Users
- **Tweets/Day**: ၅၀၀ သန်း tweets per day
- **Timeline Latency**: **< 2 seconds** for home timeline load
- **Tweet Delivery**: Follower ၅ seconds အတွင်း tweet ရရှိရမည်
- **Search Indexing**: Tweet post ပြီး **15 seconds** အတွင်း searchable ဖြစ်ရမည်
- **Trending Update**: **30 seconds** interval ဖြင့် trending topics refresh
- **Availability**: 99.9%+

### Scale Estimation (အတိုင်းအတာ ခန့်မှန်းခြင်း)

```
Tweet write QPS: 500M / 86400 = ~5,800 tweets/sec (peak 3x = 17,400)
Timeline read QPS: 250M DAU x 20 reads/day = 5B reads/day
  = ~57,870 reads/sec (Read:Write ratio ~10:1)

Storage:
  Average tweet: 200 bytes text
  500M x 200B = 100 GB/day (text only)
  With media attachments: ~1 TB/day
  10-year projection: ~3.6 PB

Fan-out challenge (အဓိက စိန်ခေါ်မှု):
  Celebrity with 100M followers posts tweet
  = 100M timeline writes needed per single tweet
  If 10 celebrities post/minute = 1 billion writes/minute!

Redis timeline memory:
  250M DAU x 1000 tweets x 16 bytes = ~4 TB Redis cluster
```

## ၄၂.၂ Domain Decomposition (ဒိုမိန်းခွဲခြမ်းခြင်း)

1. **User Service** — Account management, authentication, profile (PostgreSQL)
2. **Tweet Service** — Tweet CRUD, media attachment handling (Cassandra — high write throughput)
3. **Timeline Service** — Home timeline generation, cache management (**system ၏ core**)
4. **Follow Service** — Social graph management, follower/following counts (MySQL sharded + Redis cache)
5. **Notification Service** — Like, RT, mention, follow notifications (Cassandra + FCM/APNs push)
6. **Search Service** — Real-time tweet indexing, full-text search (Elasticsearch)
7. **Trending Service** — Sliding window trending detection (Kafka Streams + Redis)

## ၄၂.၃ Tweet Write Path — Fan-out Strategies

Home Timeline fan-out သည် Twitter system design ၏ **hardest engineering problem** ဖြစ်ပါသည်။ Fan-out strategy ၃ မျိုးကို နှိုင်းယှဉ်ပါမည်:

### Strategy 1: Fan-out on Write — Push Model (ရေးချိန်တွင် ဖြန့်ခြင်း)

Tweet post ပြုလုပ်သည်နှင့် follower အားလုံး၏ timeline cache ထဲ ချက်ချင်း ထည့်သည်:

```
Alice (100K followers) posts tweet:
  1. Write tweet to Cassandra (Tweet DB)
  2. Fan-out worker: foreach follower_id in Alice's followers:
       Redis ZADD "timeline:{follower_id}" {timestamp} {tweet_id}
  3. Followers' timelines are pre-populated

Follower reads timeline:
  ZREVRANGEBYSCORE "timeline:{user_id}" +inf -inf LIMIT 0 50
  Result: < 1ms read latency (super fast!)

Cost: 100K writes per single tweet (acceptable)
```

**Celebrity Problem** (နာမည်ကြီးသူ ပြဿနာ):
```
Elon Musk (100M followers) posts tweet:
  100M fan-out writes needed
  At 100K writes/second = 1,000 seconds = ~17 minutes delay!
  Completely unacceptable for real-time platform.
```

### Strategy 2: Fan-out on Read — Pull Model (ဖတ်ချိန်တွင် ဆွဲယူခြင်း)

Timeline ကြည့်သောအခါမှ SQL query ဖြင့် ရှာဖွေသည်:

```
User opens timeline:
  SELECT * FROM tweets
  WHERE author_id IN (SELECT following_id FROM follows WHERE user_id = ?)
  ORDER BY created_at DESC LIMIT 50

Problem: User follows 2,000 accounts
  = JOIN across 2,000 authors' tweets
  = Very slow at 500M active tweets in DB
  = Read amplification is massive (seconds per query)
```

### Strategy 3: Hybrid Approach — Twitter ၏ Actual Solution

```
Normal users (< 10M followers) --> Fan-out on Write (Push)
Celebrity users (>= 10M followers) --> Fan-out on Read (Pull)

Timeline Generation Flow:
  User opens Twitter:

  Step 1: Read pre-built timeline from Redis (pushed tweets)
    = Contains tweets from non-celebrity accounts user follows
    = O(1) Redis read, < 1ms

  Step 2: Pull celebrity tweets separately
    = For each celebrity in user's following list:
      query their recent tweets from Tweet DB (last 24 hours)
    = User follows ~5-10 celebrities typically = 5-10 DB queries

  Step 3: Merge and sort
    = Combine pushed_feed + pulled_celebrity_tweets
    = Sort by timestamp (chronological) or ML ranking score
    = Return top 50 tweets

  Total latency: < 200ms (Redis read + few DB queries + merge)
```

**Threshold determination** (ကန့်သတ်ချက် သတ်မှတ်ခြင်း):
- "Celebrity" threshold = **10 million followers**
- Background job identifies all celebrity accounts → Redis set: `"celebrities"`
- At timeline read time, check user's following against celebrities set
- Continuously re-evaluate: user crosses 10M → switch from push to pull

## ၄၂.၄ Home Timeline Architecture (Redis Timeline Cache per User)

### Redis Data Structure

```
Key: "timeline:{user_id}"
Type: Sorted Set (ZSET)
Score: tweet timestamp (Unix milliseconds)
Member: tweet_id

Operations:
  Fan-out write: ZADD timeline:{user_id} {timestamp} {tweet_id}
  Read timeline: ZREVRANGEBYSCORE timeline:{user_id} +inf -inf LIMIT 0 50
  Trim old entries: ZREMRANGEBYRANK timeline:{user_id} 0 -1001 // keep 1000

Memory per user: 1000 tweets x 16 bytes = ~16 KB
Total: 250M DAU x 16 KB = ~4 TB Redis cluster (manageable)
```

### Merging Celebrity Tweets at Read Time

```
def get_home_timeline(user_id):
    # Step 1: Get pushed timeline (non-celebrity tweets)
    pushed = redis.zrevrange(f"timeline:{user_id}", 0, 199)

    # Step 2: Get celebrity tweets (pull on read)
    celebrity_follows = get_celebrity_follows(user_id)
    pulled = []
    for celeb_id in celebrity_follows:
        recent = tweet_db.get_recent(celeb_id, hours=24, limit=20)
        pulled.extend(recent)

    # Step 3: Merge and rank
    all_tweets = merge(pushed, pulled)
    ranked = ml_ranker.rank(all_tweets, user_id)
    return ranked[:50]
```

### Cache Lifecycle

- **New user**: No cache → pull from Tweet DB (cold start) → populate Redis
- **Inactive user** (30+ days): Cache expired (TTL) → first login regenerates from DB
- **Unfollow**: Ideally remove their tweets from cache → expensive! Alternative: filter at read time using `muted:{user_id}` Redis set

## ၄၂.၅ Trending Topics (Sliding Window Counter + Kafka Streams)

### Real-Time Trending Detection Pipeline

```
Tweet posted --> Kafka "tweet-events"
                    |
                    v
        Kafka Streams Topology:

        1. Extract hashtags and keywords from each tweet
           (NLP tokenization + hashtag parsing)

        2. Tumbling Window: 1-minute count per topic
           GroupBy(topic) -> Count() -> Window(1 min)

        3. Sliding Window: rolling 1-hour trend velocity
           score = count(last_15min) / count(prev_15min)

        4. Trend Score Formula:
           trend_score = velocity x recency_weight x engagement_factor
           engagement = sum(likes + RTs + replies) per topic

        5. Top 50 trending topics --> Redis (per region)
           ZADD trending:global {score} "hashtag:WorldCup"
           ZADD trending:country:MM {score} "topic:Myanmar"
           TTL: 30 seconds (refresh every 30s)
```

### Geo-Personalization (ဒေသအလိုက် ပြသခြင်း)

```
trending:global     --> ကမ္ဘာလုံးဆိုင်ရာ trending
trending:country:US --> အမေရိကန် trending
trending:country:MM --> မြန်မာ trending
trending:city:yangon --> ရန်ကုန်မြို့ trending (volume နည်းပါက less reliable)
```

Promoted Trends (ငွေပေးကြော်ငြာ trending) ကို position 1-5 တွင် ထားပြီး organic trends ကို position 6-30 တွင် ပြသပါသည်။

## ၄၂.၆ Follow Graph Storage (Social Graph)

### Approach: MySQL Sharded Adjacency List + Graph DB Analytics

**Primary storage — MySQL (sharded)**:
```sql
CREATE TABLE follows (
  follower_id BIGINT,
  following_id BIGINT,
  created_at TIMESTAMP,
  PRIMARY KEY (follower_id, following_id),
  INDEX idx_following (following_id, follower_id)
);

-- Twitter scale: ~150 billion follow relationships
-- 150B x 16 bytes = ~2.4 TB (fits in sharded MySQL)
```

**Graph queries — Neo4j/Apache Giraph** (analytics layer):
```
-- "Friends of friends" recommendation (People You May Know)
MATCH (a:User)-[:FOLLOWS*2]->(b:User) WHERE a.id = X
RETURN b  // 2-hop traversal, very fast in graph DB

-- Mutual follows
MATCH (a)-[:FOLLOWS]->(b), (b)-[:FOLLOWS]->(a)
WHERE a.id = X RETURN b
```

**Follower count cache — Redis**:
```
INCR "followers_count:{user_id}"   // on follow
DECR "followers_count:{user_id}"   // on unfollow
```

Twitter ၏ actual implementation တွင် **FlockDB** (custom distributed graph DB built on MySQL + Gizzard sharding framework) ကို အသုံးပြုပါသည်။ ဤ design တွင် core relationships အတွက် MySQL sharded, graph analytics (PYMK — People You May Know) အတွက် Neo4j, real-time counts အတွက် Redis ကို combine ပြုလုပ်ပါသည်။

## ၄၂.၇ Tweet Search နှင့် Full-Text Indexing (Elasticsearch)

### Elasticsearch Index Schema

```json
{
  "mappings": {
    "properties": {
      "tweet_id": {"type": "keyword"},
      "user_id": {"type": "keyword"},
      "content": {"type": "text", "analyzer": "english"},
      "hashtags": {"type": "keyword"},
      "created_at": {"type": "date"},
      "lang": {"type": "keyword"},
      "like_count": {"type": "integer"},
      "rt_count": {"type": "integer"},
      "is_verified": {"type": "boolean"},
      "location": {"type": "geo_point"}
    }
  }
}
```

### Indexing Strategy

- **Time-based indices**: tweets_2026_01, tweets_2026_02 (monthly rotation)
- Older months = **read-only, compressed** (ILM — Index Lifecycle Management)
- Write path: Tweet created → Kafka → ES Indexer consumer (< 15 seconds lag)
- Read path: Search query → Elasticsearch → merge results from relevant time indices

### Search Ranking (ရှာဖွေမှု အဆင့်သတ်မှတ်ခြင်း)

```
Search: "Myanmar earthquake"

Ranking factors:
  1. Text relevance (BM25 score) — core relevance
  2. Recency (< 1 hour tweets get 3x boost)
  3. Engagement (likes + RTs weighted sum)
  4. Author credibility (verified, follower count)
  5. Personalization (author in user's following → boost)
```

## ၄၂.၈ Notification Service (Likes, RTs, Mentions)

### Event-Driven Pipeline

```
Notification Triggers --> Kafka "notification-events"
  {type: "LIKE", actor_id: "B", target_user_id: "A",
   entity_type: "tweet", entity_id: "t_123"}
          |
          v
  Notification Service Consumer:

  Step 1: Deduplication + Aggregation (5-second window)
    100 likes in 1 second --> "User B and 99 others liked your tweet"

  Step 2: Delivery
    - In-app: WebSocket push to active sessions
    - Mobile push: FCM (Android) / APNs (iOS) if app backgrounded
    - Email: Daily digest (not real-time)

  Storage: Cassandra {user_id, notif_id, type, metadata, is_read, created_at}
  Unread badge count: Redis INCR "unread:{user_id}"
```

## ၄၂.၉ Architecture Diagram

```
+------------------------------------------------------------------+
|                      TWITTER / X CLIENTS                          |
|               (Web, iOS, Android, TweetDeck)                      |
+------+----------------------------+------------------------------+
       | Tweet / Read Feed          | Search / Trending
       v                            v
+------------------------------------------------------------------+
|                  API GATEWAY (Kong / Envoy)                       |
|           Auth, Rate Limiting, Request Routing                    |
+----+--------+--------+--------+--------+--------+---------------+
     |        |        |        |        |        |
     v        v        v        v        v        v
 +------+ +------+ +------+ +------+ +------+ +------+
 |Tweet | |Time- | |Follow| |Trend-| |Search| |Notif |
 | Svc  | |line  | | Svc  | |ing   | | Svc  | | Svc  |
 +--+---+ |Svc   | +--+---+ |Svc   | +--+---+ +--+---+
    |     +--+---+    |     +--+---+    |        |
    v        v        v        |        v        v
 +------+ +------+ +------+   |   +--------+ +--------+
 |Cass- | |Redis | |MySQL |   |   |Elastic-| |Cass-   |
 |andra | |Time- | |(shard|   |   |search  | |andra   |
 |(tweet| |line  | | ed)  |   |   |(tweet  | |(notifs)|
 | store| |Cache | |+Redis|   |   | index) | |        |
 +------+ |4 TB  | |count |   |   +--------+ +--------+
          +------+ +------+   |
                               v
                    +-------------------+
                    |      KAFKA        |
                    | tweet-events      |
                    | notif-events      |
                    | trending-events   |
                    +--------+----------+
                             |
                +------------+------------+
                |            |            |
                v            v            v
          +---------+ +---------+ +------------+
          |Fan-Out  | |Notif    | |Kafka       |
          |Worker   | |Worker   | |Streams     |
          |(push to | |(aggregate| |(trending  |
          | Redis)  | | + send) | | detection)|
          +---------+ +---------+ +------------+
```

### Tweet Write + Fan-Out Data Flow

```
1. @Alice (500K followers) posts: "Hello Myanmar! #Mingalabar"

2. API Gateway -> Tweet Service:
   Save to Cassandra -> Return {tweet_id: "t_123"}

3. Kafka: "tweet-events" {tweet_id, author, follower_count: 500K}

4. Fan-Out Worker (500K < 10M threshold = Push):
   Fetch 500K follower IDs from Follow Service
   Batch ZADD to Redis timelines (~10 seconds)

5. Trending Worker: Extract "#Mingalabar"
   Increment sliding window counter
   If threshold met -> update trending:country:MM

6. Search Indexer: Index in Elasticsearch (< 15 seconds)

7. Bob (follows Alice) opens Twitter:
   Timeline Svc: ZREVRANGE timeline:bob 0 49
   Check celebrity follows -> pull their tweets
   Merge + ML rank -> return timeline (< 200ms total)
```

## Key Trade-offs (အဓိက ချိန်ညှိရမည့် အချက်များ)

| ချိန်ညှိချက် | ရွေးချယ်မှု | အကြောင်းပြချက် |
|---|---|---|
| **Fan-out Strategy** | Push-Pull Hybrid (10M threshold) | Normal users push (fast delivery), celebrities pull (avoid write storm that takes hours) |
| **Timeline Storage** | Redis Sorted Set (4 TB cluster) | Sub-millisecond read latency, sorted by timestamp natively |
| **Tweet Storage** | Cassandra | 17K writes/sec peak, time-series natural ordering, partition by user_id |
| **Social Graph** | MySQL sharded + Neo4j analytics | MySQL proven at Twitter scale (150B edges), Neo4j for graph traversal queries |
| **Search** | Elasticsearch with time-based indices | Real-time indexing (< 15s), ILM for old data compression |
| **Trending** | Kafka Streams sliding window | Real-time velocity calculation, horizontally scalable, Kafka Streams EOS + idempotent sink design ဖြင့် duplicate risk ကို လျှော့ချနိုင် |
| **Storage Cost vs Speed** | Redis 4 TB = expensive | 250M users x 16KB = 4TB Redis memory cost justified by < 1ms timeline reads serving 57K QPS |
| **Trending Window** | 30-second refresh interval | Shorter window (1s) = too noisy, longer window (5min) = too slow for breaking news |

ဤ system design တွင် **hybrid fan-out** pattern သည် core innovation ဖြစ်ပြီး normal users (99% of accounts) အတွက် push model ၏ speed နှင့် celebrity accounts (1%) အတွက် pull model ၏ write efficiency ကို combine ပြုလုပ်ထားပါသည်။ Redis timeline cache ၏ 4 TB memory cost သည် significant ဖြစ်သော်လည်း 57K QPS ကို sub-millisecond latency ဖြင့် serve လုပ်နိုင်ခြင်းကြောင့် trade-off worthwhile ဖြစ်ပါသည်။ Kafka event bus သည် tweet write path, trending detection, notification pipeline, search indexing စသည့် downstream consumers အားလုံးကို decouple လုပ်ပေးပြီး system ၏ overall resilience ကို မြှင့်တင်ပါသည်။
