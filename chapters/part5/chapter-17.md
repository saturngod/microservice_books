# အခန်း ၁၇: Distributed Caching

## နိဒါန်း

Modern distributed system တွင် caching သည် performance ၏ အဓိကသော့ချက်ဖြစ်သည်။ Database query တစ်ခုသည် milliseconds ရာပေါင်းကြာနိုင်သော်လည်း cache မှ data ရယူပါက microseconds ကိုသာ ကြာသည်။ ဤ performance gap သည် user experience ကို သိသာစွာ သက်ရောက်သည်။ သို့သော် caching သည် ရိုးရှင်းသော concept ဖြစ်သော်လည်း distributed environment တွင် implement လုပ်ရာ cache stampede၊ cache invalidation၊ distributed locks ကဲ့သို့သော challenges ပေါ်ပေါက်လာသည်။ ဤအခန်းတွင် caching strategies များကို နက်ရှိုင်းစွာ လေ့လာမည်ဖြစ်သည်။

---

## ၁၇.၁ Cache Patterns: Cache-Aside, Write-Through, Write-Behind, Refresh-Ahead

### Cache-Aside (Lazy Loading)

Application သည် cache ကို explicit manage လုပ်သည် — database miss ဖြစ်သောအခါမှသာ cache populate လုပ်သည်။

```
Read Flow:
App ──── Read ──►  Cache ──── HIT ────► Return data
                  │
                  └── MISS ──► Database ──► Return data
                                     │
                               ┌─────▼─────┐
                               │  Store in │
                               │   Cache   │
                               └───────────┘
```

```javascript
class CacheAsideService {
    constructor(redisClient, database) {
        this.cache = redisClient;
        this.db = database;
        this.TTL = 3600; // 1 hour cache TTL
    }

    async getProduct(productId) {
        const cacheKey = `product:${productId}`;
        
        // Cache ကို စစ်ဆေးသည်
        const cached = await this.cache.get(cacheKey);
        if (cached) {
            // Cache hit — cache မှ return ပြန်သည်
            return JSON.parse(cached);
        }

        // Cache miss — database မှ query လုပ်သည်
        const product = await this.db.query(
            'SELECT * FROM products WHERE id = $1',
            [productId]
        );

        if (product) {
            // Cache တွင် သိမ်းသည်
            await this.cache.setex(cacheKey, this.TTL, JSON.stringify(product));
        }

        return product;
    }

    async updateProduct(productId, data) {
        // Database update လုပ်သည်
        await this.db.query(
            'UPDATE products SET name=$1, price=$2 WHERE id=$3',
            [data.name, data.price, productId]
        );

        // Cache invalidate လုပ်သည် (delete)
        await this.cache.del(`product:${productId}`);
    }
}
```

### Write-Through

Data write သောအခါ cache နှင့် database နှစ်ခုလုံးကို တပြိုင်နက် update လုပ်သည်။

```javascript
class WriteThroughCache {
    async updateProduct(productId, data) {
        // Cache နှင့် DB ကို တပြိုင်နက် update လုပ်သည်
        const [_, dbResult] = await Promise.all([
            this.cache.setex(
                `product:${productId}`,
                this.TTL,
                JSON.stringify(data)
            ),
            this.db.query(
                'UPDATE products SET data=$1 WHERE id=$2',
                [data, productId]
            )
        ]);
        return dbResult;
    }
}
```

### Write-Behind (Write-Back)

Cache ကို ချက်ချင်း write ပြီး database ကို asynchronously update လုပ်သည်။

```javascript
class WriteBehindCache {
    constructor(redisClient, database) {
        this.cache = redisClient;
        this.db = database;
        // dirty records ကို track လုပ်သည်
        this.dirtyKeys = new Set();

        // 5 seconds တစ်ကြိမ် flush to database
        setInterval(() => this.flushToDB(), 5000);
    }

    async updateProduct(productId, data) {
        const key = `product:${productId}`;
        
        // Cache ကို ချက်ချင်း update လုပ်သည်
        await this.cache.setex(key, this.TTL, JSON.stringify(data));
        
        // dirty ဟု mark လုပ်သည်
        this.dirtyKeys.add(productId);

        // ချက်ချင်း return ပြန်သည် — DB write ကို မစောင့်ဘဲ
        return { success: true };
    }

    async flushToDB() {
        // Dirty records များကို DB တွင် persist လုပ်သည်
        for (const productId of this.dirtyKeys) {
            const data = JSON.parse(await this.cache.get(`product:${productId}`));
            if (data) {
                await this.db.query(
                    'UPDATE products SET data=$1 WHERE id=$2',
                    [data, productId]
                );
            }
        }
        this.dirtyKeys.clear();
    }
}
```

### Refresh-Ahead

Cache ၏ TTL မကုန်မီ proactively refresh လုပ်သည်။

```javascript
class RefreshAheadCache {
    // TTL ၏ 80% ကျော်သောအခါ refresh trigger ဖြစ်သည်
    REFRESH_THRESHOLD = 0.8;

    async getWithRefreshAhead(key, ttl, fetchFn) {
        const cached = await this.cache.get(key);
        const remainingTTL = await this.cache.ttl(key);

        if (cached) {
            // TTL ၏ 20% သာ ကျန်သေးလျှင် background refresh လုပ်သည်
            if (remainingTTL < ttl * (1 - this.REFRESH_THRESHOLD)) {
                // background refresh (non-blocking)
                this.refreshInBackground(key, ttl, fetchFn);
            }
            return JSON.parse(cached);
        }

        // Cache miss — fresh data fetch လုပ်သည်
        return this.fetchAndCache(key, ttl, fetchFn);
    }

    async refreshInBackground(key, ttl, fetchFn) {
        try {
            const freshData = await fetchFn();
            await this.cache.setex(key, ttl, JSON.stringify(freshData));
        } catch (error) {
            console.error('Background refresh failed:', error);
        }
    }
}
```

---

## ၁၇.၂ Redis Deep Dive

Redis သည် distributed caching အတွက် industry standard ဖြစ်သည်။ Data structures အမျိုးမျိုးနှင့် powerful features များပါဝင်သည်။

### Redis Data Structures

```javascript
const redis = require('ioredis');
const client = new redis();

// String — basic key-value, counters, cached JSON
await client.set('user:1001', JSON.stringify({ name: 'Mg Mg' }));
await client.incr('page_views'); // atomic increment

// Hash — object fields ကို efficiently သိမ်းသည်
await client.hset('user:1001:profile', 'name', 'Mg Mg', 'email', 'mg@example.com');
const name = await client.hget('user:1001:profile', 'name');

// List — queue, stack, timeline
await client.lpush('notifications:1001', JSON.stringify({ type: 'message' }));
const notifications = await client.lrange('notifications:1001', 0, 9); // latest 10

// Set — unique members, tags, followers
await client.sadd('product:1:tags', 'electronics', 'smartphone', 'apple');
const tags = await client.smembers('product:1:tags');

// Sorted Set — leaderboards, priority queues
await client.zadd('leaderboard', 9500, 'player:A'); // score: 9500
await client.zadd('leaderboard', 8200, 'player:B');
const topPlayers = await client.zrevrange('leaderboard', 0, 9, 'WITHSCORES');
```

### Redis Pub/Sub

```javascript
// Publisher
const publisher = new redis();
async function broadcastNotification(channel, data) {
    await publisher.publish(channel, JSON.stringify(data));
}

// Subscriber
const subscriber = new redis();
await subscriber.subscribe('notifications:global');

subscriber.on('message', (channel, message) => {
    const data = JSON.parse(message);
    // connected clients ထံ forward လုပ်သည်
    websocketBroadcast(data);
});
```

### Lua Scripts — Atomic Operations

```javascript
// Lua script ဖြင့် atomic rate limiting implement လုပ်သည်
const rateLimitScript = `
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    
    local current = redis.call('GET', key)
    if current == false then
        -- ပထမဆုံး request
        redis.call('SET', key, 1)
        redis.call('EXPIRE', key, window)
        return 1
    elseif tonumber(current) < limit then
        -- Limit မကျော်သေး
        return redis.call('INCR', key)
    else
        -- Limit ကျော်သွားသည်
        return -1
    end
`;

async function checkRateLimit(userId, limit = 100, windowSecs = 60) {
    const result = await client.eval(
        rateLimitScript,
        1,                      // KEYS count
        `ratelimit:${userId}`,  // KEYS[1]
        limit,                  // ARGV[1]
        windowSecs              // ARGV[2]
    );
    return result !== -1;
}
```

---

## ၁၇.၃ Cache Stampede နှင့် Dog-Piling Prevention

Cache key တစ်ခု expire ဖြစ်သောအခါ requests များစွာ တပြိုင်နက် database ကို hit လုပ်တတ်သည်။ ဤ phenomenon ကို cache stampede ဟုခေါ်သည်။

```
Cache Stampede ဖြစ်ပုံ:

  T=0:  Cache key expires
  T=1:  10,000 requests come in → ALL cache miss
  T=2:  10,000 requests hit database → DB overwhelmed
  T=3:  DB crashes or timeouts
```

### Probabilistic Early Expiration

```javascript
class StampedePreventionCache {
    constructor(redisClient, database) {
        this.cache = redisClient;
        this.db = database;
    }

    async getWithPER(key, ttl, fetchFn) {
        const data = await this.cache.get(key);
        const remainingTTL = await this.cache.ttl(key);

        if (data && remainingTTL > 0) {
            // Probabilistic Early Recomputation
            // TTL နည်းလာသည်နှင့်အမျှ refresh probability တိုးလာသည်
            const beta = 1.0; // tuning parameter
            const delta = Math.random();

            if (delta > Math.exp(-beta * remainingTTL / ttl)) {
                return JSON.parse(data);
            }
            // Proactively refresh
        }

        return this.fetchAndCache(key, ttl, fetchFn);
    }
}
```

### Mutex Lock ဖြင့် Single Flight

```javascript
class SingleFlightCache {
    constructor(redisClient, database) {
        this.cache = redisClient;
        this.db = database;
        // In-flight requests ကို track လုပ်သည်
        this.inFlight = new Map();
    }

    async get(key, ttl, fetchFn) {
        // Cache hit check
        const cached = await this.cache.get(key);
        if (cached) return JSON.parse(cached);

        // ဤ key အတွက် in-flight request ရှိနေပြီဆိုပါက join လုပ်သည်
        if (this.inFlight.has(key)) {
            return this.inFlight.get(key);
        }

        // ပထမဆုံး request — fetch start လုပ်သည်
        const promise = this.fetchAndCache(key, ttl, fetchFn)
            .finally(() => {
                // Fetch ပြီးသောအခါ in-flight မှ ဖယ်ရှားသည်
                this.inFlight.delete(key);
            });

        // Other requests အတွက် register လုပ်သည်
        this.inFlight.set(key, promise);

        return promise;
    }

    async fetchAndCache(key, ttl, fetchFn) {
        const data = await fetchFn();
        await this.cache.setex(key, ttl, JSON.stringify(data));
        return data;
    }
}
```

---

## ၁၇.₄ Cache Invalidation Strategies

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### TTL-Based Invalidation

```javascript
// Product price ကို 5 minutes ကြာ cache လုပ်သည်
await cache.setex('product:price:1001', 300, JSON.stringify(price));
```

### Event-Driven Invalidation

```javascript
// Product update event သည် cache ကို invalidate လုပ်သည်
class ProductEventHandler {
    async handleProductUpdated(event) {
        const { productId } = event;

        // ဆက်စပ် cache keys များ invalidate လုပ်သည်
        const keysToInvalidate = [
            `product:${productId}`,
            `product:detail:${productId}`,
            `product:price:${productId}`,
            `search:products:*`  // wildcard invalidation (ဂရုတစိုက် သုံးပါ)
        ];

        // Batch delete
        await Promise.all(keysToInvalidate.map(key => {
            if (key.includes('*')) {
                return this.deletePattern(key);
            }
            return cache.del(key);
        }));
    }

    async deletePattern(pattern) {
        // SCAN ဖြင့် matching keys ရှာသည် (KEYS * သုံးခြင်းသည် production ၌ dangerous)
        let cursor = '0';
        do {
            const [nextCursor, keys] = await cache.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
            cursor = nextCursor;
            if (keys.length > 0) {
                await cache.del(...keys);
            }
        } while (cursor !== '0');
    }
}
```

---

## ၁၇.၅ Distributed Locks — Redlock Algorithm

Distributed environment တွင် critical section ကို protect လုပ်ရန် distributed lock လိုအပ်သည်။

```javascript
class RedLock {
    constructor(redisClients) {
        // Multiple Redis instances (fault tolerance အတွက်)
        this.clients = redisClients;
        this.quorum = Math.floor(redisClients.length / 2) + 1;
        this.clockDriftFactor = 0.01;
    }

    async acquire(resource, ttlMs) {
        const lockValue = generateRandomString(32);
        const startTime = Date.now();

        let successCount = 0;

        // Redis instances အားလုံးတွင် lock ရယူကြိုးစားသည်
        for (const client of this.clients) {
            const success = await this.tryLock(client, resource, lockValue, ttlMs);
            if (success) successCount++;
        }

        const elapsed = Date.now() - startTime;
        const validity = ttlMs - elapsed - (ttlMs * this.clockDriftFactor);

        // Quorum (majority) ရရှိပြီး validity ကျန်နေပါက lock ရသည်
        if (successCount >= this.quorum && validity > 0) {
            return { acquired: true, lockValue, validity };
        }

        // Lock မရ — acquired locks ကို release လုပ်သည်
        await this.releaseAll(resource, lockValue);
        return { acquired: false };
    }

    async tryLock(client, resource, value, ttlMs) {
        // SET NX PX — key မရှိမှသာ set လုပ်သည်
        const result = await client.set(
            `lock:${resource}`,
            value,
            'NX',      // Only set if not exists
            'PX',      // Expire in milliseconds
            ttlMs
        );
        return result === 'OK';
    }

    async release(client, resource, lockValue) {
        // Lock value စစ်ဆေးပြီးမှသာ delete လုပ်သည် (atomic)
        const releaseScript = `
            if redis.call("GET", KEYS[1]) == ARGV[1] then
                return redis.call("DEL", KEYS[1])
            else
                return 0
            end
        `;
        await client.eval(releaseScript, 1, `lock:${resource}`, lockValue);
    }
}

// Usage
async function processPayment(orderId, amount) {
    const lock = await redlock.acquire(`order:${orderId}`, 30000);

    if (!lock.acquired) {
        throw new Error('Could not acquire lock — another process is working');
    }

    try {
        // Critical section — payment process လုပ်သည်
        await chargePayment(orderId, amount);
    } finally {
        // Lock ကို release လုပ်သည်
        await redlock.release(`order:${orderId}`, lock.lockValue);
    }
}
```

---

## ၁၇.၆ CDN Caching — Edge vs Origin

CDN (Content Delivery Network) caching သည် static content ကို users နှင့် နီးကပ်သော edge servers တွင် cache လုပ်သည်။

```
Without CDN:
User (မြန်မာ) ────────────────► Origin Server (US)
                  ~300ms latency

With CDN:
User (မြန်မာ) ──► Edge (Singapore) ──► (if miss) ──► Origin (US)
                  ~30ms latency           ~300ms only on cache miss

CDN Cache Flow:
1. User requests image.jpg
2. Edge checks local cache
3. Cache HIT → return immediately (~30ms)
4. Cache MISS → forward to origin (~300ms), cache for next requests
```

### CDN Cache Headers

```javascript
// Express.js တွင် CDN cache headers သတ်မှတ်သည်
app.get('/api/products', (req, res) => {
    // CDN တွင် 1 hour cache, browser တွင် 5 minutes cache
    res.setHeader('Cache-Control', 'public, max-age=300, s-maxage=3600');
    // Conditional requests အတွက် ETag
    res.setHeader('ETag', calculateETag(products));

    res.json(products);
});

// Static assets — aggressive caching
app.use('/static', express.static('public', {
    maxAge: '1y',                    // 1 year browser cache
    setHeaders: (res, path) => {
        // Immutable — content hash ပြောင်းမည်မဟုတ်
        res.setHeader('Cache-Control', 'public, max-age=31536000, immutable');
    }
}));

// Dynamic content — no CDN cache
app.get('/api/user/profile', authMiddleware, (req, res) => {
    // Private — CDN မှ cache မလုပ်ရ
    res.setHeader('Cache-Control', 'private, no-cache');
    res.json(userProfile);
});
```

### Cache Purging Strategy

```javascript
// Cloudflare CDN cache purge
async function purgeProductCache(productId) {
    const purgeUrls = [
        `https://api.example.com/products/${productId}`,
        `https://www.example.com/products/${productId}`,
        `https://cdn.example.com/images/product-${productId}.jpg`
    ];

    await fetch(`https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/purge_cache`, {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${CF_API_TOKEN}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({ files: purgeUrls })
    });
}
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

- **Cache Patterns:** Cache-Aside သည် most flexible ဖြစ်ပြီး lazy loading approach ဖြစ်သည်; Write-Through သည် read-heavy workloads အတွက် ကောင်းသည်; Write-Behind သည် write performance ကို optimize လုပ်သော်လည်း data loss risk ရှိသည်; Refresh-Ahead သည် cache miss ကို minimize လုပ်သည်
- **Redis Data Structures:** String, Hash, List, Set, Sorted Set တို့ကို data type အပေါ်မူတည်၍ ရွေးချယ်သုံးစွဲပြီး Lua scripts ဖြင့် atomic operations implement လုပ်နိုင်သည်
- **Cache Stampede:** Single Flight pattern ဖြင့် duplicate requests ကို collapse လုပ်ပြီး Probabilistic Early Expiration ဖြင့် stampede ကို proactively ကာကွယ်နိုင်သည်
- **Cache Invalidation:** TTL-based နှင့် event-driven invalidation နှစ်မျိုးလုံး combine လုပ်ပြီး SCAN command ကို KEYS * ၏ production-safe alternative ဖြစ်သောကြောင့် wildcard deletion တွင် အသုံးပြုသင့်သည်
- **Redlock Algorithm:** Multiple Redis instances (quorum ≥ 3) ဖြင့် distributed lock implement လုပ်ပြီး critical sections ကို protect ရမည်; lock value စစ်ဆေးပြီးမှ release ဖြင့် accidental unlock ကာကွယ်သင့်သည်
- **CDN Caching:** Static assets တွင် aggressive immutable caching ကို သုံးပြီး dynamic content တွင် appropriate Cache-Control headers သတ်မှတ်ကာ product updates ဖြစ်သောအခါ targeted cache purging ကို implement ရမည်
