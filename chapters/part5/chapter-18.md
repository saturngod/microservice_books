# အခန်း ၁၈: Sharding, Partitioning နှင့် Replication

## နိဒါန်း

Data volume ကြီးလာသည်နှင့်အမျှ single database server တစ်ခုတည်းဖြင့် handle မလုပ်နိုင်သောအချိန် ရောက်ရှိလာသည်။ ဤသည် horizontal scaling ၏ တိုက်ရိုက်ပြဿနာဖြစ်ပြီး sharding နှင့် partitioning ကို ဖြေရှင်းနည်းအဖြစ် ကျင့်သုံးသည်။ Sharding ဆိုသည်မှာ data ကို multiple database nodes များတွင် split ဖြန့်ကြဲပြီး replication ဆိုသည်မှာ data copies များကို multiple nodes တွင် ထိန်းသိမ်းကာ availability နှင့် read performance ကို မြှင့်တင်ခြင်းဖြစ်သည်။ ဤအခန်းတွင် ဤ concepts များကို deeply နားလည်မည်ဖြစ်သည်။

---

## ၁၈.၁ Horizontal vs Vertical Scaling

```
Vertical Scaling (Scale Up):
┌─────────────────────┐         ┌──────────────────────────┐
│  Server             │  ───►   │  Bigger Server           │
│  4 CPU, 16GB RAM    │         │  64 CPU, 512GB RAM        │
│  1TB SSD            │         │  10TB NVMe               │
└─────────────────────┘         └──────────────────────────┘
  ↑ ကန့်သတ်ချက်ရှိသည် (physical limit)
  ↑ Cost exponentially တိုးသည်
  ↑ Single point of failure

Horizontal Scaling (Scale Out):
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  DB Node 1   │    │  DB Node 2   │    │  DB Node 3   │
│  Shard A     │    │  Shard B     │    │  Shard C     │
│  Users A-M   │    │  Users N-R   │    │  Users S-Z   │
└──────────────┘    └──────────────┘    └──────────────┘
  ↑ Theoretically unlimited
  ↑ Cost linear
  ↑ No single point of failure
```

---

## ၁၈.၂ Sharding Strategies

### Hash Sharding

```javascript
class HashShardRouter {
    constructor(shards) {
        this.shards = shards;        // shard connections array
        this.shardCount = shards.length;
    }

    // User ID ကို shard သို့ map လုပ်သည်
    getShardIndex(userId) {
        // Consistent hash function
        const hash = this.murmurHash(userId.toString());
        return hash % this.shardCount;
    }

    getShard(userId) {
        const index = this.getShardIndex(userId);
        return this.shards[index];
    }

    async getUser(userId) {
        // မှန်ကန်သော shard ကို route လုပ်သည်
        const shard = this.getShard(userId);
        return shard.query('SELECT * FROM users WHERE id = $1', [userId]);
    }

    async createUser(userId, userData) {
        const shard = this.getShard(userId);
        return shard.query(
            'INSERT INTO users (id, data) VALUES ($1, $2)',
            [userId, userData]
        );
    }

    // Simple hash function implementation
    murmurHash(str) {
        let hash = 0;
        for (let i = 0; i < str.length; i++) {
            const char = str.charCodeAt(i);
            hash = ((hash << 5) - hash) + char;
            hash = hash & hash; // 32-bit integer
        }
        return Math.abs(hash);
    }
}
```

### Range Sharding

```
Range Sharding Example (By User ID):

Shard 1: User IDs 1 - 1,000,000
Shard 2: User IDs 1,000,001 - 2,000,000
Shard 3: User IDs 2,000,001 - 3,000,000

┌─────────────────────────────────────────────────┐
│              Routing Table                       │
├───────────────────┬──────────────────────────────┤
│ Range             │ Shard                        │
├───────────────────┼──────────────────────────────┤
│ 1 - 1,000,000     │ db-shard-1.internal          │
│ 1M+1 - 2,000,000  │ db-shard-2.internal          │
│ 2M+1 - 3,000,000  │ db-shard-3.internal          │
└───────────────────┴──────────────────────────────┘
```

```javascript
class RangeShardRouter {
    constructor() {
        // Range → shard mapping
        this.ranges = [
            { min: 1, max: 1000000, shard: 'db-shard-1' },
            { min: 1000001, max: 2000000, shard: 'db-shard-2' },
            { min: 2000001, max: 3000000, shard: 'db-shard-3' },
        ];
    }

    getShardForId(id) {
        const range = this.ranges.find(r => id >= r.min && id <= r.max);
        if (!range) throw new Error(`No shard found for id: ${id}`);
        return range.shard;
    }
}
```

### Directory Sharding

```javascript
class DirectoryShardRouter {
    constructor(metadataDB, cache) {
        this.metadataDB = metadataDB;
        this.cache = cache;
    }

    async getShardForTenant(tenantId) {
        // Cache မှ lookup လုပ်သည်
        const cached = await this.cache.get(`shard:tenant:${tenantId}`);
        if (cached) return cached;

        // Metadata DB မှ shard mapping ရယူသည်
        const result = await this.metadataDB.query(
            'SELECT shard_id FROM tenant_shard_mapping WHERE tenant_id = $1',
            [tenantId]
        );

        const shardId = result.rows[0]?.shard_id;
        if (shardId) {
            await this.cache.setex(`shard:tenant:${tenantId}`, 3600, shardId);
        }

        return shardId;
    }

    // Tenant ကို shard assignment ပြောင်းသည် (rebalancing)
    async moveTenantToShard(tenantId, newShardId) {
        await this.metadataDB.query(
            'UPDATE tenant_shard_mapping SET shard_id=$1 WHERE tenant_id=$2',
            [newShardId, tenantId]
        );
        // Cache invalidate လုပ်သည်
        await this.cache.del(`shard:tenant:${tenantId}`);
    }
}
```

---

## ၁၈.၃ Consistent Hashing နှင့် Virtual Nodes

Traditional hashing ၏ ပြဿနာမှာ node ထပ်ထည့်ခြင်း/ဖယ်ရှားခြင်းသောအခါ data redistribution ကြီးမားသည်။ Consistent hashing ဖြင့် ဤပြဿနာကို ဖြေရှင်းနိုင်သည်။

```
Consistent Hash Ring:

                    0°
                  User:5
              /            \
        315°                  45°
      Node C    ──────      Node A
        |      Hash Ring      |
      270°                  90°
      Node B    ──────      Node D
              \            /
                  180°
                User:8

Virtual Nodes (Vnodes):
Node A = [A1, A2, A3, A4, A5] — 5 virtual nodes on the ring
Node B = [B1, B2, B3, B4, B5] — distributes load more evenly
```

```javascript
class ConsistentHashRing {
    constructor(nodes = [], virtualNodeCount = 100) {
        this.virtualNodeCount = virtualNodeCount;
        // Sorted ring
        this.ring = new Map();
        this.sortedKeys = [];

        // Initial nodes ထည့်သည်
        nodes.forEach(node => this.addNode(node));
    }

    addNode(node) {
        // Virtual nodes ဖန်တီးသည်
        for (let i = 0; i < this.virtualNodeCount; i++) {
            const vKey = this.hash(`${node}:vnode:${i}`);
            this.ring.set(vKey, node);
            this.sortedKeys.push(vKey);
        }
        this.sortedKeys.sort((a, b) => a - b);
    }

    removeNode(node) {
        for (let i = 0; i < this.virtualNodeCount; i++) {
            const vKey = this.hash(`${node}:vnode:${i}`);
            this.ring.delete(vKey);
        }
        this.sortedKeys = this.sortedKeys.filter(k => this.ring.has(k));
    }

    getNode(key) {
        if (this.sortedKeys.length === 0) return null;

        const keyHash = this.hash(key);

        // Hash ၏ clockwise direction တွင် ရှိသော ပထမဆုံး node ရှာသည်
        const index = this.sortedKeys.findIndex(k => k >= keyHash);

        // Wrap around — ring ၏ ထိပ်သို့ ပြန်သွားသည်
        const nodeKey = this.sortedKeys[index === -1 ? 0 : index];
        return this.ring.get(nodeKey);
    }

    hash(key) {
        // FNV-1a hash function
        let hash = 2166136261;
        for (let i = 0; i < key.length; i++) {
            hash ^= key.charCodeAt(i);
            hash = (hash * 16777619) >>> 0;
        }
        return hash;
    }
}

// Usage
const ring = new ConsistentHashRing(['node1', 'node2', 'node3']);
console.log(ring.getNode('user:1001')); // node2
console.log(ring.getNode('user:1002')); // node1

// Node ထည့်သောအခါ data ၏ small portion ကိုသာ migrate လုပ်ရသည်
ring.addNode('node4');
```

---

## ၁၈.₄ Read Replicas နှင့် Leader Election

### Read Replica Architecture

```
Write Operations → Leader (Primary)
Read Operations → Replicas (Secondary)

┌─────────────────────────────────────────────────┐
│                    Clients                       │
└─────┬──────────────────────────────┬────────────┘
      │ Writes                        │ Reads
      ▼                               ▼
┌─────────────┐              ┌───────────────────┐
│   Leader    │──replicate──►│   Replica 1       │
│  (Primary)  │──replicate──►│   Replica 2       │
│             │──replicate──►│   Replica 3       │
└─────────────┘              └───────────────────┘
```

```javascript
class DatabaseCluster {
    constructor(leader, replicas) {
        this.leader = leader;
        this.replicas = replicas;
        this.currentReplicaIndex = 0;
    }

    // Writes သည် Leader သို့ သွားသည်
    async write(query, params) {
        return this.leader.query(query, params);
    }

    // Reads သည် Replicas သို့ round-robin ဖြင့် သွားသည်
    async read(query, params) {
        const replica = this.getNextReplica();
        try {
            return await replica.query(query, params);
        } catch (error) {
            // Replica fail → fallback to leader
            console.warn('Replica failed, falling back to leader');
            return this.leader.query(query, params);
        }
    }

    getNextReplica() {
        // Round-robin load balancing
        const replica = this.replicas[this.currentReplicaIndex];
        this.currentReplicaIndex = (this.currentReplicaIndex + 1) % this.replicas.length;
        return replica;
    }
}
```

### Leader Election (Raft Algorithm Simplified)

```
Leader Election Process:

1. All nodes start as FOLLOWER
2. If no heartbeat from leader → become CANDIDATE
3. CANDIDATE requests votes from other nodes
4. If majority votes received → become LEADER
5. LEADER sends heartbeats to maintain leadership

States:
FOLLOWER ──(timeout)──► CANDIDATE ──(majority votes)──► LEADER
    ▲                        │                              │
    └────────────────────────┘                              │
          (higher term discovered)          heartbeats      │
    ◄────────────────────────────────────────────────────────┘
```

---

## ၁၈.₅ Hot Spot Problem နှင့် Rebalancing

### Hot Spot Detection

```javascript
class HotSpotMonitor {
    constructor(shards) {
        this.shards = shards;
        // query count per shard ကို track လုပ်သည်
        this.shardMetrics = new Map();
    }

    recordQuery(shardId) {
        const metrics = this.shardMetrics.get(shardId) || { count: 0, lastReset: Date.now() };
        metrics.count++;
        this.shardMetrics.set(shardId, metrics);
    }

    detectHotSpots(threshold = 10000) {
        const hotSpots = [];

        for (const [shardId, metrics] of this.shardMetrics.entries()) {
            const qps = metrics.count / ((Date.now() - metrics.lastReset) / 1000);

            if (qps > threshold) {
                hotSpots.push({ shardId, qps });
                console.warn(`Hot spot detected: Shard ${shardId} — ${qps} QPS`);
            }
        }

        return hotSpots;
    }
}
```

### Shard Splitting

```
Hot Shard Split:

Before:
Shard 1: Users 1-2,000,000 (overloaded — 50,000 QPS)

After Split:
Shard 1a: Users 1-1,000,000    (25,000 QPS each)
Shard 1b: Users 1,000,001-2,000,000

Migration Steps:
1. Create Shard 1b
2. Copy data for Users 1M+1 to 2M
3. Update routing table
4. Stop writes to old range
5. Final sync
6. Switch traffic
```

---

## ၁၈.₆ Global Databases — CockroachDB နှင့် Spanner

### CockroachDB — Distributed SQL

```sql
-- CockroachDB — globally distributed SQL database
-- Multi-region table configuration
CREATE TABLE orders (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL,
    region      VARCHAR(20) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT now()
) LOCALITY REGIONAL BY ROW;

-- Data ကို region-specific nodes တွင် store လုပ်သည်
-- Myanmar users ၏ data → APAC region nodes

-- Global table — all regions တွင် replicate လုပ်သည်
CREATE TABLE config (
    key   VARCHAR(255) PRIMARY KEY,
    value TEXT
) LOCALITY GLOBAL;
```

### Google Spanner Architecture

```
Spanner Architecture:

                    ┌─────────────────────────────┐
                    │     TrueTime (Atomic Clock) │
                    └─────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌─────────┐     ┌─────────┐     ┌─────────┐
        │  Zone   │     │  Zone   │     │  Zone   │
        │  US-E   │     │  EU-W   │     │ APAC    │
        └────┬────┘     └────┬────┘     └────┬────┘
             │               │               │
        ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
        │ Spanserv│     │ Spanserv│     │ Spanserv│
        │ tablets │     │ tablets │     │ tablets │
        └─────────┘     └─────────┘     └─────────┘
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

- **Sharding Strategies:** Hash sharding သည် uniform data distribution ပေးသော်လည်း range queries ကို efficient မလုပ်နိုင်; Range sharding သည် range queries ကောင်းသော်လည်း hot spot ဖြစ်နိုင်; Directory sharding သည် most flexible ဖြစ်သော်လည်း extra metadata overhead ရှိသည်
- **Consistent Hashing:** Node ထပ်ထည့်ခြင်း/ဖယ်ရှားခြင်းသောအခါ data ၏ small fraction ကိုသာ remapping လုပ်ရသောကြောင့် traditional modulo hashing ထက် သာလွန်သည်; Virtual nodes ဖြင့် load distribution ကို ပိုမိုသမာသည်
- **Read Replicas:** Write operations ကို leader တစ်ခုတည်းသို့ route ပြီး read operations ကို multiple replicas တွင် spread လုပ်ခြင်းဖြင့် read throughput ကို linear scale လုပ်နိုင်သည်
- **Hot Spots:** Query pattern monitoring ဖြင့် hot shards ကို early detect ပြီး shard splitting ဖြင့် load ကို ပြန်ဖြန့်နိုင်သည်; Consistent hashing ဖြင့် rebalancing impact ကို minimize လုပ်နိုင်သည်
- **Global Databases:** CockroachDB နှင့် Google Spanner ကဲ့သို့သော globally distributed databases များသည် multi-region deployments တွင် automatic failover နှင့် global consistency ပေးဆောင်သောကြောင့် complex distributed system management burden ကို လျှော့ချသည်
