# နောက်ဆက်တွဲ A: System Design ခန့်မှန်းချက် အမြန်ကိုးကားကဒ်

---

## A.1 မှတ်ဉာဏ်အရွယ်အစား ကူးပြောင်းဇယား

| ယူနစ် | တန်ဖိုး | ဥပမာ |
|--------|---------|-------|
| 1 Bit | 0 သို့မဟုတ် 1 | Boolean တစ်ခု |
| 1 Byte (B) | 8 bits | ASCII စာလုံးတစ်လုံး |
| 1 Kilobyte (KB) | 1,024 Bytes ≈ 10³ B | ငယ်သော text ဖိုင်တစ်ခု |
| 1 Megabyte (MB) | 1,024 KB ≈ 10⁶ B | MP3 သီချင်းတစ်ပုဒ် (ပျမ်းမျှ) |
| 1 Gigabyte (GB) | 1,024 MB ≈ 10⁹ B | HD ရုပ်ရှင်တစ်ကား |
| 1 Terabyte (TB) | 1,024 GB ≈ 10¹² B | HDD disk drive တစ်ခု |
| 1 Petabyte (PB) | 1,024 TB ≈ 10¹⁵ B | ကြီးမားသော data center storage |
| 1 Exabyte (EB) | 1,024 PB ≈ 10¹⁸ B | Global internet traffic (monthly) |
| 1 Zettabyte (ZB) | 1,024 EB ≈ 10²¹ B | အားလုံးသော digital data (ကမ္ဘာပေါ်) |

---

## A.2 2 ၏ အထပ်ထပ်မြှောက် ကိုးကားဇယား (Powers of 2)

| Power | တန်ဖိုး | ခန့်မှန်းတန်ဖိုး | အမည် |
|-------|---------|----------------|------|
| 2¹⁰ | 1,024 | ≈ 1 thousand | 1 KB |
| 2²⁰ | 1,048,576 | ≈ 1 million | 1 MB |
| 2³⁰ | 1,073,741,824 | ≈ 1 billion | 1 GB |
| 2³² | 4,294,967,296 | ≈ 4 billion | IPv4 addresses အများဆုံးအရေအတွက် |
| 2⁴⁰ | ~1.1 trillion | ≈ 1 trillion | 1 TB |
| 2⁵⁰ | ~1.1 quadrillion | ≈ 1 quadrillion | 1 PB |
| 2⁶⁴ | ~1.8 × 10¹⁹ | ≈ 18 quintillion | unsigned 64-bit int အများဆုံး |

---

## A.3 Latency နံပါတ်များ — Engineer တိုင်းသိထားသင့်သည်

| လုပ်ဆောင်ချက် | Latency | မှတ်ချက် |
|--------------|---------|---------|
| L1 cache reference | 0.5 ns | အမြန်ဆုံး CPU cache |
| Branch misprediction | 5 ns | CPU pipeline flush |
| L2 cache reference | 7 ns | L1 ထက် 14x နှေး |
| Mutex lock/unlock | 25 ns | Threading overhead |
| Main memory (RAM) reference | 100 ns | L1 ထက် 200x နှေး |
| Compress 1 KB with Zippy | 3 μs (3,000 ns) | — |
| Send 2 KB over 1 Gbps network | 20 μs | — |
| Read 1 MB sequentially from RAM | 250 μs | — |
| Round trip in same datacenter | 500 μs | LAN round trip |
| SSD random read | 100 μs | NVMe SSD |
| Read 1 MB sequentially from SSD | 1 ms | — |
| HDD seek time | 10 ms | Spinning disk |
| Read 1 MB sequentially from HDD | 20 ms | — |
| Send packet CA→Netherlands→CA | 150 ms | Cross-continent |

> **အရေးကြီးသောနိဂုံး:** RAM သည် disk ထက် ~1,000x မြန်သည်။ Network round trip (cross-continent) သည် disk seek ထက် ~15x နှေးသည်။

---

## A.4 အချိန် ကိုးကားဇယား

| ကာလ | စက္ကန့်အရေအတွက် | ခန့်မှန်းတန်ဖိုး |
|-----|----------------|----------------|
| 1 မိနစ် | 60 | — |
| 1 နာရီ | 3,600 | ≈ 3.6 × 10³ |
| 1 ရက် | 86,400 | ≈ 10⁵ (100,000) |
| 1 ပတ် | 604,800 | ≈ 6 × 10⁵ |
| 1 လ (30 ရက်) | 2,592,000 | ≈ 2.5 × 10⁶ |
| 1 နှစ် | 31,536,000 | ≈ 3 × 10⁷ |
| 2.5 နှစ် | ~10⁸ seconds | မှတ်ရလွယ်သော မှတ်ချက် |

> **လက်တွေ့သုံး:** QPS တွက်ချက်ရန် — တစ်ရက် request **86,400** ဖြင့် စားပါ။

---

## A.5 QPS (Queries Per Second) ခန့်မှန်းချက် Template

```
QPS = Total Requests Per Day / 86,400

Peak QPS = Average QPS × 2  (သို့မဟုတ် × 3 ဆိုလျှင် conservative ဖြစ်သည်)
```

### တွက်နည်း ဥပမာ

| Metric | တွက်ချက်ချက် | ရလဒ် |
|--------|------------|------|
| DAU (Daily Active Users) | — | 10 million |
| တစ်ဦးချင်း တစ်ရက် read requests | — | 10 |
| Total reads/day | 10M × 10 | 100M |
| Average Read QPS | 100M / 86,400 | ~1,157 QPS |
| Peak Read QPS | 1,157 × 2 | ~2,314 QPS |

---

## A.6 Storage ခန့်မှန်းချက် Template

```
Storage Per Day = QPS × Payload Size (bytes) × 86,400

5-Year Storage = Storage Per Day × 365 × 5
```

### ဥပမာ — Tweet storage

| Item | တန်ဖိုး |
|------|---------|
| Tweet တစ်ခု ပျမ်းမျှ ဆိုဒ် | 300 bytes (text + metadata) |
| Tweets per day | 500 million |
| Storage per day | 500M × 300B = **150 GB/day** |
| Storage per year | 150 × 365 ≈ **54 TB/year** |
| 5-year storage | ≈ **270 TB** |

---

## A.7 Bandwidth ခန့်မှန်းချက် Template

```
Bandwidth = QPS × Average Response Size

Inbound bandwidth  = Write QPS × Write Payload Size
Outbound bandwidth = Read QPS  × Read Payload Size
```

### ဥပမာ — Video Streaming

| Item | တန်ဖိုး |
|------|---------|
| Concurrent viewers | 1 million |
| Video bitrate (1080p) | 5 Mbps |
| Total outbound bandwidth | 1M × 5 Mbps = **5 Tbps** |

---

## A.8 ကမ္ဘာ့ကျော် Platform Scale Benchmarks

| Platform | Metric |
|----------|--------|
| **Twitter / X** | ~500 million tweets/day; Peak QPS ~6,000 |
| **YouTube** | 500 hours of video uploaded/minute; 1 billion hours watched/day |
| **Uber** | ~14 million trips/day; peak ~200 trips/second |
| **WhatsApp** | ~100 billion messages/day; 2 billion+ users |
| **Instagram** | ~1 billion photos/day processed; ~95 million posts/day |
| **Netflix** | ~15% of global internet bandwidth at peak; 220M+ subscribers |
| **Amazon** | ~4,000 orders/second on Prime Day peak |
| **Google Search** | ~8.5 billion searches/day (~100,000 QPS) |
| **Facebook** | ~100 billion messages/day (Messenger + WhatsApp) |
| **TikTok** | ~1 billion MAU; 1B videos viewed/day |

---

## A.9 ကောင်းမွန်သော Back-of-envelope တွက်ချက်မှုများ လမ်းညွှန်

### အဆင့် ၁ — Clarify Requirements

- **DAU / MAU** — သုံးစွဲသူ ဘယ်နှစ်ယောက်?
- **Read : Write ratio** — ဘယ်ဘက် ပိုများသည်?
- **Data retention** — ဘယ်လောက်ကြာ သိမ်းဆည်းမည်?

### အဆင့် ၂ — Key Numbers

```
1. QPS (average + peak)
2. Storage per day / year
3. Bandwidth (inbound + outbound)
4. Cache size (20% hot data rule of thumb)
```

### အဆင့် ၃ — Sanity Check

- တွက်ချက်မှုများသည် မျှတရဲ့လား?
- Bottleneck ဘာဖြစ်သည်? (CPU, RAM, Disk, Network)
- Single machine ဖြင့် handle နိုင်မလား? မနိုင်လျှင် — sharding / distribution လိုသည်

---

## A.10 ဘုံ Rule of Thumbs

| Rule | အသုံးပြုမှု |
|------|-----------|
| 1 server ≈ 10,000 QPS (stateless API) | Horizontal scaling ကိန်းဂဏန်း |
| Cache hit rate ≥ 80% ရည်မှန်း | Cache efficiency target |
| Hot data ≈ 20% of total data | Pareto principle (80/20 rule) |
| P99 latency ≤ 1s for user-facing APIs | User experience threshold |
| 3 replicas minimum for HA | High availability standard |
| RPO < 1 hour for critical data | Backup frequency guideline |
| Network packet loss < 0.1% | Production network standard |
