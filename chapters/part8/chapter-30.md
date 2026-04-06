# အခန်း ၃၀: Search Engine (Google / Elasticsearch-Based)

## နိဒါန်း

Search engine သည် internet ၏ most complex distributed systems ထဲမှ တစ်ခုဖြစ်သည်။ Google သည် တစ်နေ့လျှင် 8.5 billion searches ကို process ပြုလုပ်ပြီး billions of web pages ကို index ပြုသည်။ ဤအခန်းတွင် web crawler architecture, inverted index construction, ranking algorithms, typeahead service, real-time indexing, နှင့် personalization တို့ကို end-to-end microservices system design perspective မှ လေ့လာမည်ဖြစ်သည်။

Search engine system ကို သုံးပိုင်းခွဲ၍ ကြည့်နိုင်သည်: **Crawl** (web ကို discover ပြုလုပ်ခြင်း) → **Index** (content ကို structured format ဖြင့် store ပြုလုပ်ခြင်း) → **Serve** (user queries ကို relevant results ဖြင့် respond ပြုလုပ်ခြင်း)။ ဤ pipeline တစ်ခုစီတွင် unique engineering challenges ရှိသည်။

---

## ၃၀.၁ Requirements နှင့် Scale Estimation

### Functional Requirements

- **Web Crawling**: Internet ၏ web pages များကို continuously discover, fetch, parse ပြုလုပ်ရမည်
- **Indexing**: Crawled content ကို searchable format ဖြင့် store ပြုလုပ်ရမည်
- **Search**: User queries ကို relevant results ဖြင့် < 200ms ဖြင့် respond ပြုရမည်
- **Typeahead**: User typing ပြုသောအချိန်တွင် real-time suggestions ပေးရမည်
- **Ranking**: Results ကို relevance + quality score ဖြင့် rank ပြုရမည်
- **Personalization**: User history ကို based on results customize ပြုရမည်

### Scale Estimation

```
Web Scale:
  - Total indexed pages: 100 billion
  - New/updated pages/day: 10 billion (10% of index/day)
  - Average page size: 20 KB (HTML, compressed)
  - Total index storage: 100B × 20 KB = 2 PB (raw)
  - After compression: ~500 TB

Crawl:
  - Pages to crawl/day: 10 billion
  - 10B / 86,400s ≈ 115,740 pages/second
  - With 10ms/page: Need ~1,157 crawler workers

Search:
  - Queries/day: 8.5 billion
  - 8.5B / 86,400s ≈ 98,380 QPS
  - Peak: ~300,000 QPS

Typeahead:
  - 3× search QPS (every keystroke)
  - ~300,000 typeahead requests/second
```

---

## ၃၀.၂ Web Crawler Architecture

### Crawl Architecture Components

```
┌──────────────────────────────────────────────────────────────────┐
│                        URL Frontier                               │
│         Priority Queue: [Priority, URL, Domain, Last-Crawl]      │
│         Backend: Kafka (distributed) + Redis (priority score)    │
└────────────────────┬─────────────────────────────────────────────┘
                     │
         ┌───────────▼──────────────┐
         │     Crawler Workers      │
         │  (1,000+ parallel)       │
         │  - DNS resolution        │
         │  - robots.txt check      │
         │  - HTTP fetch            │
         │  - HTML parse            │
         │  - URL extraction        │
         └───────────┬──────────────┘
                     │
    ┌────────────────▼───────────────────┐
    │         Content Processor          │
    │  - Duplicate detection (SimHash)   │
    │  - Language detection              │
    │  - Content extraction              │
    │  - Link extraction + normalization │
    └────────────────┬───────────────────┘
                     │
         ┌───────────▼──────────────┐
         │    Content Store (S3)    │
         │  Raw HTML + Extracted    │
         └───────────┬──────────────┘
                     │
         ┌───────────▼──────────────┐
         │     Index Pipeline       │
         │  (Indexing Service)      │
         └──────────────────────────┘
```

### Breadth-First Crawling

```
Seed URLs: ["https://wikipedia.org", "https://bbc.com", ...]

BFS Algorithm:
  Queue: [wikipedia.org, bbc.com, ...]
  Visited: Set() (Bloom Filter for 100B URLs)

  while queue not empty:
    url = queue.dequeue()
    if url in visited: continue
    visited.add(url)
    
    html = fetch(url)
    links = extract_links(html)
    for link in links:
      if link not in visited:
        queue.enqueue(link, priority=calculate_priority(link))
```

### Politeness Policy

```
Rules:
  1. robots.txt compliance:
     - Fetch /robots.txt before crawling domain
     - Cache robots.txt for 24 hours
     - Respect Crawl-delay directive

  2. Rate limiting per domain:
     - Max 1 request/second per domain (polite)
     - Max 10 requests/second per domain (aggressive, with permission)

  3. Courtesy delay:
     - Domain last crawl time + delay before re-crawl

robots.txt example:
  User-agent: *
  Disallow: /private/
  Crawl-delay: 2

  User-agent: Googlebot
  Allow: /
  Crawl-delay: 0.5
```

### Re-crawl Strategy

```
Re-crawl Priority:
  1. Change frequency (via HTTP Last-Modified / ETag)
  2. PageRank (high authority pages crawled more often)
  3. Content change signals (sitemap.xml, ping APIs)

Re-crawl Schedule:
  High priority (news, social media): Every 5 minutes
  Medium priority (blog posts): Every 1-7 days
  Low priority (static pages): Every 30 days
  Archive pages: Every 6 months

Conditional fetch:
  GET /page HTTP/1.1
  If-None-Match: "etag-value-123"
  If-Modified-Since: Mon, 01 Apr 2026 00:00:00 GMT

  Response: 304 Not Modified → No re-index needed (bandwidth saving)
```

---

## ၃၀.၃ Inverted Index Construction

### Forward Index vs Inverted Index

```
Forward Index (Doc → Terms):
  doc1: {words: ["apple", "mango", "fruit"]}
  doc2: {words: ["apple", "pie", "recipe"]}
  doc3: {words: ["mango", "juice"]}

Inverted Index (Term → Docs):
  "apple"  → [(doc1, pos:0, tf:1), (doc2, pos:0, tf:1)]
  "mango"  → [(doc1, pos:1, tf:1), (doc3, pos:0, tf:1)]
  "fruit"  → [(doc1, pos:2, tf:1)]
  "pie"    → [(doc2, pos:1, tf:1)]
  "recipe" → [(doc2, pos:2, tf:1)]

Search "apple":
  Lookup "apple" in index → [doc1, doc2] (fast!)
```

### Index Structure

```
Inverted Index Entry:
{
  term: "machine_learning",
  document_frequency: 5000000,  // # docs containing term
  postings: [
    {
      doc_id: 12345,
      term_frequency: 3,     // how many times in doc
      positions: [45, 78, 102],  // for phrase search
      field: "title"         // title/body/anchor weight
    },
    ...
  ]
}

Compressed with: Variable-Byte Encoding + Delta compression
  Raw postings: 1 TB → Compressed: ~200 GB
```

### Index Build Pipeline

```
Crawled HTML → 
  Text Extraction (remove HTML tags) →
  Tokenization (split into tokens) →
  Normalization (lowercase, stemming: "running" → "run") →
  Stop word removal ("the", "is", "at" removed) →
  N-gram generation (bigrams for phrase search) →
  Posting list construction →
  Sorting by term →
  Index segment writing (immutable Lucene segments)

MapReduce/Spark Job:
  Map: (doc_id, html) → [(term, doc_id, tf, pos)]
  Reduce: (term) → sorted postings list
```

---

## ၃၀.၄ Ranking Algorithms

### TF-IDF (Term Frequency - Inverse Document Frequency)

```
TF(t, d) = (count of term t in doc d) / (total terms in doc d)
IDF(t) = log(N / df(t))
  N = total documents
  df(t) = documents containing term t

TF-IDF(t, d) = TF(t, d) × IDF(t)

Example:
  Query: "machine learning"
  
  doc1 (AI textbook): 
    TF("machine", doc1) = 50/1000 = 0.05
    IDF("machine") = log(1B/500M) = 0.30
    TF-IDF("machine", doc1) = 0.05 × 0.30 = 0.015

  High TF-IDF → Term is important for this document
```

### BM25 (Best Match 25) — Elasticsearch Default

BM25 သည် TF-IDF ၏ improved version ဖြစ်ပြီး term frequency saturation နှင့် document length normalization ကို handle ပြုသည်။

```
BM25(q, d) = Σ IDF(qi) × (tf(qi,d) × (k1 + 1)) / 
                          (tf(qi,d) + k1 × (1 - b + b × |d|/avgdl))

Parameters:
  k1 = 1.2 (TF saturation: prevents very high TF from dominating)
  b  = 0.75 (document length normalization)
  avgdl = average document length
```

### PageRank

```
Algorithm: Random surfer model
  - Web = directed graph (pages as nodes, links as edges)
  - PageRank(A) = probability that random surfer lands on A

PR(A) = (1-d)/N + d × Σ(PR(Ti) / C(Ti))
  d = damping factor (0.85)
  N = total pages
  Ti = pages linking to A
  C(Ti) = outbound link count of Ti

Interpretation:
  - High PR: many high-quality pages link to this page
  - Wikipedia has very high PR (many inbound links)
```

### Composite Ranking

```
final_score = α × relevance_score   // BM25, TF-IDF
            + β × quality_score     // PageRank, domain authority
            + γ × freshness_score   // Recency for news queries
            + δ × personalization   // User click history
            + ε × local_score       // Geographic relevance

Typical weights:
  α = 0.40, β = 0.30, γ = 0.15, δ = 0.10, ε = 0.05
```

---

## ၃၀.၅ Typeahead / Autocomplete Service

### Trie-based Approach

```
Trie for autocomplete:
  "a" → "apple", "amazon", "android"
  "ap" → "apple", "apple pie", "apply"
  "app" → "apple", "apple store", "appletv"
  "appl" → "apple", "apple store"
  "apple" → "apple", "apple store", "apple music"

Each node stores: {prefix, top_k_completions, score}
top_k = 5 most popular completions for this prefix

Build: From query logs (daily batch job)
  - Count query frequencies
  - Build trie with top-5 per prefix
  - Serialize to Redis (fast prefix lookups)
```

### Real-Time Query Serving

```
GET /autocomplete?q=appl&limit=5

Redis: GET autocomplete:appl
→ Cache HIT: ["apple", "apply", "apple store", "application", "appliance"]
→ Return in < 5ms

Cache MISS:
  - Trie lookup: O(M) where M = prefix length
  - Return top-K completions sorted by score
  - Cache result for 1 hour

Scoring:
  completion_score = query_frequency × recency_factor × ctr_score
  (CTR = Click-Through Rate from past results)
```

---

## ၃၀.၆ Index Sharding နှင့် Near-Real-Time Indexing

### Horizontal Index Sharding

```
Sharding Strategies:

1. Document-based Sharding (Random):
   doc_shard = hash(doc_id) % num_shards
   Pros: Even distribution
   Cons: Query broadcasts to all shards

2. Term-based Sharding:
   term_shard = hash(term) % num_shards
   Pros: Efficient for single-term queries
   Cons: Multi-term queries complex

3. Topic/Category Sharding:
   Sports docs → Sports shards
   News docs → News shards
   Pros: Semantic isolation

Production (Elasticsearch approach):
  - Primary shards: 5 (based on data size)
  - Replica shards: 1 per primary
  - Total shards: 10
  - Routing: doc_id based (deterministic)
```

### Near-Real-Time (NRT) Indexing

```
Lucene Segment Architecture:
  
  New document arrives:
    → Write to in-memory buffer (IndexWriter buffer)
    → Buffer full OR refresh interval → Create new segment
    → Segment becomes searchable (NRT refresh, default: 1 second)
    → Background merge: many small segments → fewer large segments

Timeline:
  T+0ms: Document written to in-memory buffer
  T+1s: Refresh → Segment created → Document searchable
  T+10s: Segment merged with others (background)

Lucene Merge Policy:
  TieredMergePolicy: Merge segments of similar size
  After merge: Delete old segments
```

### Write-Ahead Log (WAL) for Durability

```
Document arrives:
  1. Write to WAL (Kafka/transaction log)
  2. Write to in-memory buffer
  3. ACK to client

On crash:
  1. Replay WAL from last checkpoint
  2. Reconstruct in-memory buffer
  3. Resume indexing
```

---

## ၃၀.၇ Personalization နှင့် Safe Search

### Personalization Signals

```
User Model:
{
  "user_id": "user-123",
  "interests": ["technology", "programming", "AI"],
  "recent_queries": ["python tutorial", "machine learning"],
  "click_history": [
    {"url": "docs.python.org", "query": "python", "timestamp": "..."},
    ...
  ],
  "location": "Yangon, MM",
  "language": "my",
  "safe_search": "moderate"
}

Personalization at Serving Time:
  base_results = search(query)
  user_model = get_user_model(user_id)
  
  for result in base_results:
    if result.topic in user_model.interests:
      result.score *= 1.2  // Boost relevant topics
    if result.domain in user_model.preferred_domains:
      result.score *= 1.1  // Boost preferred sources
  
  re-rank by adjusted scores
```

### Safe Search Filters

```
Content Classification (ML model):
  - Explicit content detection (image + text)
  - Violence/harm detection
  - Spam/malware detection

Safe Search Levels:
  OFF: No filtering
  MODERATE: Filter explicit images, keep borderline text
  STRICT: Filter all adult content

Implementation:
  - Pre-filtered index segments per safe search level
  - Query-time filter: +safe_search_level:moderate
  - Or document-level tags in index
```

---

## ၃၀.၈ Architecture Diagram နှင့် Data Flow

### Full System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                          Internet                                 │
│                    (100 Billion Web Pages)                        │
└───────────────────────┬──────────────────────────────────────────┘
                        │ Crawl
┌───────────────────────▼──────────────────────────────────────────┐
│                    Crawl Pipeline                                  │
│                                                                    │
│  URL Frontier → Crawler Workers → Content Processor → Store (S3)  │
│  (Kafka Queue)   (1000+ nodes)    (Parse + Dedup)   (PB-scale)   │
└───────────────────────┬──────────────────────────────────────────┘
                        │ Raw Content
┌───────────────────────▼──────────────────────────────────────────┐
│                    Index Pipeline (Spark/Flink)                    │
│                                                                    │
│  Tokenize → Normalize → TF-IDF Score → Build Inverted Index       │
│  → Shard by doc_id → Write to Index Cluster                       │
└───────────────────────┬──────────────────────────────────────────┘
                        │ Index data
┌───────────────────────▼──────────────────────────────────────────┐
│                  Search Index Cluster                              │
│        (Elasticsearch / Custom Lucene-based)                       │
│                                                                    │
│  Shard 0  Shard 1  Shard 2  ...  Shard N                         │
│  (Replica) (Replica) (Replica)   (Replica)                       │
└───────────────────────┬──────────────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────────────┐
│                    Search Serving Layer                            │
│                                                                    │
│  ┌─────────────────┐  ┌─────────────┐  ┌────────────────────┐   │
│  │  Query Service  │  │  Ranking    │  │  Personalization   │   │
│  │  (Parse +       │  │  Service    │  │  Service           │   │
│  │   Rewrite)      │  │  (BM25+PR)  │  │  (User Model)      │   │
│  └────────┬────────┘  └──────┬──────┘  └────────────────────┘   │
│           └─────────────────┬┘                                   │
│                             │                                     │
│  ┌──────────────────────────▼────────────────────────────────┐   │
│  │              Typeahead / Autocomplete Service              │   │
│  │                    (Redis Trie)                            │   │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                        │
                 ┌──────▼──────┐
                 │    Client   │
                 │  (Browser)  │
                 └─────────────┘
```

### Query Processing Data Flow

```
1. User types "machine learning tutorial" → Typeahead suggestions (Redis)
2. User submits query
3. Query Service:
   a. Spell check: "mahcine" → "machine"
   b. Query expansion: "ml tutorial" added
   c. Intent detection: Educational content
   d. Safe search filter applied

4. Index Cluster: Broadcast query to all shards
   - Each shard: Local BM25 scoring → top-K local results
   - Merge coordinator: Merge + global rank

5. Ranking Service:
   - Apply PageRank boost
   - Apply freshness boost (for tutorial queries: recent preferred)
   - Apply personalization scores

6. User model lookup (if logged in): Boost familiar topics
7. Return top-10 results with snippets in < 200ms
```

---

## Key Design Decisions နှင့် Trade-offs

| Challenge | Solution | Trade-off |
|---|---|---|
| 100B pages | Sharded Lucene index | Complexity vs Performance |
| Real-time crawl | Kafka-based URL frontier | Durability vs Simplicity |
| Dedup detection | SimHash (near-duplicate) | CPU cost vs Accuracy |
| Ranking | BM25 + PageRank composite | Tuning complexity vs Quality |
| Typeahead latency | Redis Trie cache | Memory cost vs Speed |
| Personalization | User model + reranking | Privacy vs Relevance |
| NRT indexing | Lucene segments (1s refresh) | Freshness vs Throughput |

## အနှစ်ချုပ်

Search engine system ၏ engineering complexity မှာ web-scale data (petabytes) ကို real-time query latency (< 200ms) ဖြင့် serve ပြုလုပ်ရခြင်းဖြစ်သည်။ Crawl pipeline, inverted index, ranking algorithms, typeahead service တို့ကို separate microservices အဖြစ် decompose ပြုလုပ်ခြင်းဖြင့် ၎င်းတို့ကို independently scale နှင့် optimize ပြုနိုင်သည်။ BM25 + PageRank ၏ composite ranking သည် text relevance နှင့် link authority ကို balance ပြုလုပ်ပေးပြီး personalization layer ဖြင့် individual user experience ကို improve ပြုနိုင်သည်။ Search system တည်ဆောက်ရာတွင် most critical challenge မှာ freshness (NRT indexing) နှင့် throughput ကြားတွင် ၊ နှင့် relevance ကို maintain ပြုရင်း scalability ကို achieve ပြုရခြင်းဖြစ်သည်။
