# အခန်း ၁၉: Search Infrastructure (ရှာဖွေမှု အခြေခံအဆောက်အအုံ)

## နိဒါန်း

Microservices architecture တွင် data များကို ထိရောက်စွာ ရှာဖွေနိုင်ရန်အတွက် Search Infrastructure သည် အလွန်အရေးကြီးသော အစိတ်အပိုင်းတစ်ခု ဖြစ်ပါသည်။ E-commerce site တစ်ခုတွင် ပစ္စည်းပေါင်း သန်းချီ ရှိသည့်အခါ၊ Social media platform တွင် post ပေါင်းများစွာ ရှိသည့်အခါ၊ Log management system တွင် log entries ဘီလီယံချီ ရှိသည့်အခါ - ဤ data အားလုံးကို millisecond အတွင်း ရှာဖွေပေးနိုင်ရမည်။ ပုံမှန် database ၏ `LIKE '%keyword%'` query သည် ဤ scale တွင် လုံးဝ မလုံလောက်ပါ။ Full-text search engine တစ်ခု လိုအပ်ပါသည်။ ဤအခန်းတွင် Elasticsearch ၏ architecture မှစ၍ full-text search, data syncing, relevance tuning နှင့် autocomplete တို့ကို အသေးစိတ် လေ့လာပါမည်။

---

## ၁၉.၁ Elasticsearch Architecture (Index, Shard, Replica)

Elasticsearch သည် Apache Lucene ပေါ်တွင် တည်ဆောက်ထားသော distributed search engine တစ်ခု ဖြစ်ပါသည်။ JSON documents များကို သိမ်းဆည်းပြီး near real-time search ပေးစွမ်းနိုင်သည်။ ၎င်း၏ core architecture ကို နားလည်ရန် အောက်ပါ diagram ကို ကြည့်ပါ။

```
+-----------------------------------------------------------+
|                  Elasticsearch Cluster                      |
|                                                             |
|  +------------------+  +------------------+  +------------+ |
|  |     Node 1       |  |     Node 2       |  |   Node 3   | |
|  |  (Master)        |  |                  |  |            | |
|  |                  |  |                  |  |            | |
|  | [Shard 0-Primary]|  | [Shard 1-Primary]|  | [Shard 2-P]| |
|  | [Shard 1-Replica]|  | [Shard 2-Replica]|  | [Shard 0-R]| |
|  +------------------+  +------------------+  +------------+ |
+-----------------------------------------------------------+
```

### အဓိက သဘောတရားများ

- **Cluster** - Elasticsearch nodes အုပ်စုတစ်ခု ဖြစ်ပြီး cluster name ဖြင့် ခွဲခြားသည်။ Node များအားလုံးသည် data ကို မျှဝေပြီး ပူးပေါင်း ရှာဖွေနိုင်သည်။
- **Node** - Cluster အတွင်းရှိ server instance တစ်ခု ဖြစ်သည်။ Master node, data node, ingest node, coordinating node စသည့် role များ ခွဲခြားနိုင်သည်။
- **Index** - Relational database ရှိ database/table နှင့် ဆင်တူသည်။ Documents များကို သိမ်းဆည်းထားသော logical namespace ဖြစ်သည်။
- **Shard** - Index တစ်ခုကို ပိုင်းခြားထားသော Lucene instance တစ်ခု ဖြစ်သည်။ Horizontal scaling အတွက် အသုံးဝင်သည်။
- **Replica** - Primary shard ၏ မိတ္တူဖြစ်ပြီး high availability နှင့် read throughput တိုးမြှင့်ရန် အသုံးပြုသည်။

```python
# Elasticsearch index ဖန်တီးခြင်း - shard နှင့် replica သတ်မှတ်ခြင်း
from elasticsearch import Elasticsearch

# Elasticsearch client ချိတ်ဆက်ခြင်း
es = Elasticsearch(["http://localhost:9200"])

# Index settings - shard ၃ ခု၊ replica ၂ ခု သတ်မှတ်ခြင်း
index_settings = {
    "settings": {
        "number_of_shards": 3,      # Primary shard အရေအတွက်
        "number_of_replicas": 2      # Replica အရေအတွက် (primary တစ်ခုလျှင်)
    },
    "mappings": {
        "properties": {
            "title": {"type": "text"},          # Full-text search အတွက်
            "category": {"type": "keyword"},     # Exact match/aggregation အတွက်
            "price": {"type": "float"},          # ကိန်းဂဏန်းအတွက်
            "description": {"type": "text"},     # အရှည်စာသားအတွက်
            "created_at": {"type": "date"}       # ရက်စွဲအတွက်
        }
    }
}

# Index ဖန်တီးခြင်း
es.indices.create(index="products", body=index_settings)
print("Index created successfully")  # Index ဖန်တီးပြီးကြောင်း
```

**အရေးကြီးသော မှတ်ချက်** - Primary shard အရေအတွက်ကို index ဖန်တီးပြီးနောက် ပြောင်းလဲ၍ မရပါ (reindex လုပ်ရပါမည်)။ ထို့ကြောင့် capacity planning ကို ဦးစွာ စနစ်တကျ လုပ်ဆောင်ရပါမည်။ Shard တစ်ခုလျှင် 20-50GB အကြား ထားရှိရန် အကြံပြုပါသည်။

---

## ၁၉.၂ Inverted Index & Full-Text Search

Elasticsearch ၏ အဓိက အားသာချက်မှာ **Inverted Index** data structure ဖြစ်ပါသည်။ ပုံမှန် database တွင် document -> words (forward index) ဟု ရှာသော်လည်း inverted index တွင် word -> documents ဟု ပြောင်းပြန် ရှာဖွေပါသည်။ ဤနည်းဖြင့် keyword တစ်ခုပါဝင်သော documents အားလုံးကို O(1) time complexity ဖြင့် ရှာတွေ့နိုင်ပါသည်။

```
+------------------------------------------------------------------+
|                      Inverted Index                               |
+------------------------------------------------------------------+
|  Term            | Document IDs          | Positions              |
|------------------|-----------------------|------------------------|
|  "microservice"  | [doc_1, doc_3, doc_7] | [5, 1, 12]            |
|  "docker"        | [doc_2, doc_3, doc_8] | [1, 8, 3]             |
|  "kubernetes"    | [doc_1, doc_5, doc_7] | [9, 2, 1]             |
|  "container"     | [doc_2, doc_3, doc_5] | [3, 15, 7]            |
+------------------------------------------------------------------+
```

### Analyzer Pipeline

Text ကို index မလုပ်မီ analyzer pipeline မှ ဖြတ်သန်းရပါသည်။ ဤ pipeline သည် text ကို searchable tokens အဖြစ် ပြောင်းလဲပေးသည်။

```
Input Text: "Running Docker Containers!"
        |
   [Character Filter]  --> "running docker containers"   (HTML strip, pattern replace)
        |
   [Tokenizer]         --> ["running", "docker", "containers"]  (whitespace split)
        |
   [Token Filter]      --> ["run", "docker", "contain"]  (stemming, lowercase, stop words)
```

```python
# Custom analyzer ဖန်တီးခြင်း
custom_analyzer_settings = {
    "settings": {
        "analysis": {
            "analyzer": {
                # မြန်မာဘာသာနှင့် အင်္ဂလိပ်ဘာသာ နှစ်မျိုးလုံး ရှာဖွေနိုင်သော analyzer
                "my_custom_analyzer": {
                    "type": "custom",
                    "tokenizer": "standard",
                    "filter": ["lowercase", "english_stemmer", "english_stop"]
                }
            },
            "filter": {
                "english_stemmer": {
                    "type": "stemmer",
                    "language": "english"   # အင်္ဂလိပ် stemming - "running" -> "run"
                },
                "english_stop": {
                    "type": "stop",
                    "stopwords": "_english_"  # "the", "a", "is" တို့ကို ဖယ်ရှားခြင်း
                }
            }
        }
    }
}

# Full-text search query ဥပမာ - Bool query ဖြင့် ရှုပ်ထွေးသော ရှာဖွေမှု
query = {
    "query": {
        "bool": {
            "must": [
                # title field တွင် "microservice" ပါဝင်ရမည်
                {"match": {"title": "microservice architecture"}}
            ],
            "should": [
                # description တွင် "scalable" ပါလျှင် score ပိုမြင့်မည်
                {"match": {"description": "scalable distributed"}}
            ],
            "filter": [
                # စျေးနှုန်း ၁၀၀ အောက်ဖြစ်ရမည် (score မထိခိုက်)
                {"range": {"price": {"lte": 100}}},
                # category တိကျစွာ ကိုက်ညီရမည်
                {"term": {"category": "technology"}}
            ],
            "must_not": [
                # "deprecated" ပါဝင်သော ရလဒ်များကို ဖယ်ရှားခြင်း
                {"match": {"title": "deprecated"}}
            ]
        }
    },
    "highlight": {
        "fields": {"title": {}, "description": {}}  # ကိုက်ညီသော စာသားကို highlight လုပ်ခြင်း
    }
}

# ရှာဖွေခြင်း လုပ်ဆောင်ခြင်း
results = es.search(index="products", body=query)
for hit in results["hits"]["hits"]:
    print(f"Score: {hit['_score']}, Title: {hit['_source']['title']}")
```

---

## ၁၉.၃ Syncing Data from Primary DB to Elasticsearch (CDC)

Primary database (PostgreSQL, MySQL) မှ Elasticsearch သို့ data ကို sync လုပ်ရန် **Change Data Capture (CDC)** pattern ကို အသုံးပြုပါသည်။ Primary DB သည် source of truth ဖြစ်ပြီး Elasticsearch သည် search-optimized secondary store ဖြစ်သည်။

```
+------------+       +----------+       +---------+       +---------------+
| PostgreSQL | ----> | Debezium | ----> |  Kafka  | ----> | Kafka Connect |
| (Primary)  |  WAL  | (CDC)    |       | Topic   |       | ES Sink       |
+------------+       +----------+       +---------+       +------+--------+
                                                                  |
                                                                  v
                                                          +---------------+
                                                          | Elasticsearch |
                                                          | (Search)      |
                                                          +---------------+
```

### CDC ချဉ်းကပ်နည်း ၃ မျိုး နှိုင်းယှဉ်ခြင်း

| ချဉ်းကပ်နည်း | Latency | Complexity | Consistency |
|---------------|---------|------------|-------------|
| **Polling-based** | မြင့်သည် (seconds-minutes) | နိမ့်သည် | ကောင်းသည် |
| **Log-based (Debezium)** | နိမ့်သည် (ms-seconds) | အလယ်အလတ် | အကောင်းဆုံး |
| **Dual Write** | နိမ့်ဆုံး | နိမ့်သည် | ပြဿနာရှိနိုင်သည် |

**Log-based CDC** သည် production တွင် အသုံးအများဆုံး ဖြစ်ပါသည်။ Database ၏ transaction log (PostgreSQL WAL, MySQL binlog) ကို ဖတ်ခြင်းဖြင့် ပြောင်းလဲမှုတိုင်းကို capture လုပ်သည်။

```python
# Kafka consumer - CDC event များကို Elasticsearch သို့ sync လုပ်ခြင်း
from kafka import KafkaConsumer
from elasticsearch import Elasticsearch, helpers
import json

es = Elasticsearch(["http://localhost:9200"])

# Kafka consumer ဖန်တီးခြင်း - Debezium CDC topic ကို subscribe လုပ်ခြင်း
consumer = KafkaConsumer(
    'dbserver1.public.products',            # Debezium topic naming convention
    bootstrap_servers=['localhost:9092'],
    group_id='es-sync-consumer',            # Consumer group ID
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    auto_offset_reset='earliest'            # အစမှ ဖတ်ခြင်း
)

# Batch processing အတွက် buffer
batch = []
BATCH_SIZE = 100  # တစ်ကြိမ်လျှင် ၁၀၀ ခု bulk index လုပ်ခြင်း

for message in consumer:
    event = message.value
    payload = event.get('payload', event)
    operation = payload['op']  # 'c'=create, 'u'=update, 'd'=delete, 'r'=read(snapshot)

    if operation in ('c', 'u', 'r'):
        # Document ကို index/update လုပ်ခြင်း
        doc = payload['after']
        batch.append({
            "_op_type": "index",
            "_index": "products",
            "_id": doc['id'],
            "_source": doc
        })
    elif operation == 'd':
        # Document ကို ဖျက်ခြင်း
        doc_id = payload['before']['id']
        batch.append({
            "_op_type": "delete",
            "_index": "products",
            "_id": doc_id
        })

    # Batch size ပြည့်လျှင် bulk index လုပ်ခြင်း
    if len(batch) >= BATCH_SIZE:
        helpers.bulk(es, batch)  # Bulk API - performance ပိုကောင်းသည်
        batch = []
        consumer.commit()        # Offset commit လုပ်ခြင်း
```

---

## ၁၉.၄ Relevance Tuning (BM၂၅, Vector Search)

Search results များ၏ အရည်အသွေးကို မြှင့်တင်ရန် relevance tuning သည် အရေးကြီးပါသည်။ "ကောင်းမွန်သော search" ဆိုသည်မှာ user ရှာလိုသော ရလဒ်ကို ပထမဆုံး ပြသပေးနိုင်ခြင်း ဖြစ်သည်။

### BM25 Algorithm

Elasticsearch သည် default အနေဖြင့် **BM25 (Best Match 25)** scoring algorithm ကို အသုံးပြုပါသည်။ ယခင် TF-IDF ၏ ပိုမိုကောင်းမွန်သော version ဖြစ်ပြီး document length normalization ပါဝင်သည်။

```
BM25 Score = IDF(q) * (tf(q,D) * (k1 + 1)) / (tf(q,D) + k1 * (1 - b + b * |D|/avgdl))

tf(q,D) = term frequency - document D တွင် query term q ပေါ်သော အကြိမ်ရေ
IDF(q)  = inverse document frequency - term q ၏ ရှားပါးမှု (ရှားလေ score မြင့်လေ)
|D|     = document D ၏ အရှည်
avgdl   = collection ရှိ documents များ၏ ပျမ်းမျှ အရှည်
k1      = 1.2 (default) - term frequency saturation parameter
b       = 0.75 (default) - length normalization parameter
```

```python
# Field boosting ဖြင့် relevance tuning လုပ်ခြင်း
boosted_query = {
    "query": {
        "multi_match": {
            "query": "microservice design patterns",
            "fields": [
                "title^3",          # title field ကို ၃ ဆ boost လုပ်ခြင်း
                "description^1.5",  # description ကို ၁.၅ ဆ boost လုပ်ခြင်း
                "content"           # content ကို default weight ထားခြင်း
            ],
            "type": "best_fields",  # အကောင်းဆုံး ကိုက်ညီသော field ၏ score ကို ယူခြင်း
            "fuzziness": "AUTO"     # Typo ခံနိုင်ရည် ရှိခြင်း
        }
    },
    # Function score - popularity ဖြင့် score ချိန်ညှိခြင်း
    "functions": [{
        "field_value_factor": {
            "field": "popularity",  # လူကြိုက်များမှု score
            "modifier": "log1p",    # Logarithmic scaling
            "factor": 0.5
        }
    }]
}
```

### Vector Search (Semantic Search)

BM25 သည် keyword matching ပဲ လုပ်နိုင်သော်လည်း Vector Search (semantic search) သည် အဓိပ္ပာယ်တူညီသော ရလဒ်များကို ပြန်ပေးနိုင်ပါသည်။ ဥပမာ "car" ဟု ရှာလျှင် "automobile", "vehicle" ပါဝင်သော ရလဒ်များကိုပါ ပြန်ပေးနိုင်သည်။

```python
# Vector search အတွက် index mapping
vector_mapping = {
    "mappings": {
        "properties": {
            "title": {"type": "text"},
            "title_vector": {
                "type": "dense_vector",  # Vector field သတ်မှတ်ခြင်း
                "dims": 768,             # BERT embedding dimension
                "index": True,
                "similarity": "cosine"   # Cosine similarity အသုံးပြုခြင်း
            }
        }
    }
}

# Hybrid search - BM25 နှင့် Vector search ပေါင်းစပ်ခြင်း
hybrid_query = {
    "query": {
        "bool": {
            "should": [
                # BM25 keyword search
                {"match": {"title": {"query": "microservice patterns", "boost": 0.3}}},
            ]
        }
    },
    # KNN vector search
    "knn": {
        "field": "title_vector",
        "query_vector": [0.12, -0.34, 0.56, 0.78],  # Query embedding vector
        "k": 10,                                       # ရလဒ် ၁၀ ခု ပြန်ပေးခြင်း
        "num_candidates": 100,                         # HNSW algorithm candidate count
        "boost": 0.7                                   # Vector score weight
    }
}
```

---

## ၁၉.၅ Autocomplete & Type-Ahead at Scale

အသုံးပြုသူက စာလုံးအချို့ ရိုက်ထည့်လိုက်ရုံဖြင့် suggestion များ ပြသပေးသော autocomplete feature သည် modern search experience ၏ မရှိမဖြစ် အစိတ်အပိုင်း ဖြစ်ပါသည်။

```
User types: "mic"  (keystroke တိုင်းတွင် query ပို့ခြင်း)
            |
   +--------v-----------+
   |  Debounce (300ms)   |   <-- Network call လျှော့ချခြင်း
   +--------+------------+
            |
   +--------v-----------+
   |  Autocomplete API   |
   +--------+------------+
            |
   Suggestions:
   +- "microservices architecture"  (popularity: 9500)
   +- "microsoft azure"             (popularity: 8200)
   +- "microphone setup"            (popularity: 3100)
```

### Completion Suggester (FST-based)

Elasticsearch ၏ completion suggester သည် Finite State Transducer (FST) data structure ကို အသုံးပြုပြီး memory ထဲတွင် prefix lookup ကို O(prefix_length) time ဖြင့် လုပ်ဆောင်နိုင်သည်။

```python
# Completion suggester mapping
suggest_mapping = {
    "mappings": {
        "properties": {
            "title": {"type": "text"},
            "title_suggest": {
                "type": "completion",     # FST data structure သုံးသည်
                "contexts": [{
                    "name": "category",
                    "type": "category"    # Category context ဖြင့် filter လုပ်နိုင်ခြင်း
                }]
            }
        }
    }
}

# Document index လုပ်ခြင်း - weight ဖြင့် priority သတ်မှတ်ခြင်း
doc = {
    "title": "Microservices Design Patterns",
    "title_suggest": {
        "input": ["microservices", "design patterns", "microservices design"],
        "weight": 95,              # Popularity weight - မြင့်လေ ဦးစားပေးလေ
        "contexts": {
            "category": ["technology"]
        }
    }
}
es.index(index="suggestions", body=doc)

# Autocomplete query - fuzzy matching ဖြင့်
suggest_query = {
    "suggest": {
        "product_suggest": {
            "prefix": "mic",              # အသုံးပြုသူ ရိုက်ထည့်သော စာသား
            "completion": {
                "field": "title_suggest",
                "size": 5,                # ရလဒ် ၅ ခု ပြန်ပေးခြင်း
                "fuzzy": {
                    "fuzziness": "AUTO"   # Typo ခံနိုင်ရည် - "micr" -> "micro"
                },
                "contexts": {
                    "category": [{        # Technology category ကိုသာ ရှာခြင်း
                        "context": "technology",
                        "boost": 2
                    }]
                }
            }
        }
    }
}
```

**Scale အတွက် optimization နည်းလမ်းများ** - Completion suggester ကို Redis cache (TTL 5-10 minutes) နှင့် တွဲသုံးခြင်း၊ client side debouncing (200-300ms)၊ edge n-gram tokenizer ဖြင့် search-as-you-type field type ကို အသုံးပြုခြင်းတို့ ဖြစ်သည်။

---

## ၁၉.၆ Elasticsearch vs OpenSearch vs Typesense

Search engine ရွေးချယ်ရာတွင် project ၏ လိုအပ်ချက်ပေါ်မူတည်၍ မတူညီသော ရွေးချယ်မှုများ ရှိပါသည်။

| Feature | Elasticsearch | OpenSearch | Typesense |
|---------|--------------|------------|-----------|
| License | SSPL/Elastic License | Apache 2.0 | GPL v3 |
| Cluster Management | ရှုပ်ထွေးသည် | ရှုပ်ထွေးသည် | ရိုးရှင်းသည် |
| Vector Search | ရှိသည် (kNN) | ရှိသည် (kNN) | ရှိသည် |
| Typo Tolerance | Plugin/custom လိုသည် | Plugin/custom လိုသည် | Built-in |
| Setup Complexity | မြင့်သည် | မြင့်သည် | နိမ့်သည် |
| Community | အကြီးဆုံး | ကြီးသည် (AWS backed) | တိုးတက်လာသည် |
| Best For | Large-scale enterprise | AWS ecosystem | Small-medium apps |
| Geo Search | ရှိသည် | ရှိသည် | ရှိသည် |
| ML Features | ရှိသည် (paid) | ရှိသည် (free) | မရှိသေး |

**ရွေးချယ်ရန် လမ်းညွှန်**

- **Elasticsearch** - Enterprise-grade system၊ complex analytics/aggregation လိုအပ်သည့်အခါ၊ Elastic ecosystem (Kibana, Logstash, Beats) ကို အပြည့်အဝ အသုံးပြုလိုသည့်အခါ
- **OpenSearch** - AWS ပေါ်တွင် run နေပြီး open-source license လိုအပ်သည့်အခါ၊ Elasticsearch ၏ license ပြောင်းလဲမှုကြောင့် migrate လုပ်လိုသည့်အခါ
- **Typesense** - Startup သို့မဟုတ် typo tolerance အရေးကြီးသော app၊ setup လွယ်ကူမှု ဦးစားပေးသည့်အခါ၊ developer experience ကောင်းမွန်မှု လိုအပ်သည့်အခါ

---

## အဓိက အချက်များ (Key Takeaways)

- Elasticsearch cluster ကို shard နှင့် replica များဖြင့် horizontal scaling လုပ်နိုင်ပြီး shard sizing (20-50GB/shard) ကို ဂရုတစိုက် plan ချရမည်
- Inverted Index သည် full-text search ၏ အခြေခံ data structure ဖြစ်ပြီး analyzer pipeline ဖြင့် text ကို searchable tokens အဖြစ် ပြောင်းလဲသည်
- CDC (Debezium + Kafka) သည် primary DB မှ search engine သို့ real-time sync လုပ်ရန် production-grade နည်းလမ်း ဖြစ်ပြီး dual write ထက် consistency ပိုကောင်းသည်
- BM25 သည် keyword search အတွက်၊ Vector Search သည် semantic search အတွက် သင့်လျော်ပြီး Hybrid search ဖြင့် နှစ်မျိုးပေါင်းစပ် အသုံးပြုနိုင်သည်
- Autocomplete အတွက် completion suggester (FST) နှင့် Redis caching ကို တွဲသုံးပါ
- Search engine ရွေးချယ်ရာတွင် scale, license, operational complexity, ecosystem တို့ကို ထည့်သွင်းစဉ်းစားပါ
