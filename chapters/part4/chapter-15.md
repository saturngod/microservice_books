# အခန်း ၁၅: Server-Sent Events နှင့် Streaming APIs

## နိဒါန်း

Real-time data delivery ၏ ရိုးရှင်းသော်လည်း အစွမ်းထက်သော approach တစ်ခုမှာ Server-Sent Events (SSE) ဖြစ်သည်။ WebSocket ၏ complexity မလိုဘဲ server မှ client သို့ data stream ပေးပို့နိုင်သည်။ Stock price updates၊ news feeds၊ progress notifications — ဤကဲ့သို့သော one-directional streaming scenarios တွင် SSE သည် elegant solution တစ်ခုဖြစ်သည်။ ဤအခန်းတွင် SSE protocol ကို နက်ရှိုင်းစွာ လေ့လာပြီး gRPC streaming နှင့် comparison ပြုလုပ်မည်ဖြစ်သည်။

---

## ၁၅.၁ SSE Protocol နှင့် Browser Support

Server-Sent Events သည် HTML5 specification ၏ အစိတ်အပိုင်းဖြစ်ပြီး `text/event-stream` MIME type ကို အသုံးပြုသည်။

### SSE Event Format

SSE message format သည် plain text ဖြစ်ပြီး အလွန်ရိုးရှင်းသည်:

```
data: ဤသည် simple message ဖြစ်သည်\n\n

event: userJoined\n
data: {"userId": "123", "name": "မောင်မောင်"}\n\n

id: 42\n
event: priceUpdate\n
data: {"symbol": "AAPL", "price": 150.25}\n\n

: ဤသည် comment ဖြစ်သည် (heartbeat အတွက် သုံးနိုင်သည်)\n\n
```

**SSE Field Types:**
- `data:` — message payload
- `event:` — custom event type
- `id:` — message ID (reconnection အတွက် အသုံးဝင်သည်)
- `retry:` — reconnection delay (milliseconds)
- `: ` — comment/heartbeat

### Browser တွင် SSE ချိတ်ဆက်ခြင်း

```javascript
// Browser-side JavaScript
const eventSource = new EventSource('/api/stream');

// default 'message' event
eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log('Message လက်ခံရသည်:', data);
    updateUI(data);
};

// custom event types listen လုပ်သည်
eventSource.addEventListener('priceUpdate', (event) => {
    const price = JSON.parse(event.data);
    document.getElementById('price').textContent = price.value;
});

// error handling
eventSource.onerror = (error) => {
    console.error('SSE connection error:', error);
    if (eventSource.readyState === EventSource.CLOSED) {
        console.log('Connection ပိတ်သွားသည်');
    }
};

// connection ပိတ်ရန်
function disconnect() {
    eventSource.close();
}
```

### Node.js SSE Server

```javascript
const express = require('express');
const app = express();

app.get('/api/stream', (req, res) => {
    // SSE headers သတ်မှတ်သည်
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.setHeader('Access-Control-Allow-Origin', '*');

    // connection ID generate လုပ်သည်
    const clientId = Date.now();
    console.log(`Client ${clientId} ချိတ်ဆက်လာသည်`);

    // helper function — SSE message ပေးပို့သည်
    const sendEvent = (eventName, data, id) => {
        if (id) res.write(`id: ${id}\n`);
        if (eventName) res.write(`event: ${eventName}\n`);
        res.write(`data: ${JSON.stringify(data)}\n\n`);
    };

    // ချက်ချင်း initial data ပေးပို့သည်
    sendEvent('connected', { clientId, message: 'Connection established' });

    // 30 seconds တစ်ကြိမ် heartbeat ပေးပို့သည်
    const heartbeat = setInterval(() => {
        res.write(': heartbeat\n\n');
    }, 30000);

    // client ချိတ်ဆက်မှု ဖြတ်သောအခါ cleanup လုပ်သည်
    req.on('close', () => {
        clearInterval(heartbeat);
        console.log(`Client ${clientId} ဆက်သွယ်မှု ကင်းသွားသည်`);
    });

    // event emitter ဖြင့် external events ကို stream လုပ်သည်
    const onNewData = (data) => {
        sendEvent('update', data, data.id);
    };

    // global event emitter မှ subscribe လုပ်သည်
    eventEmitter.on('dataUpdate', onNewData);

    req.on('close', () => {
        eventEmitter.off('dataUpdate', onNewData);
    });
});
```

### Browser Support

```
┌─────────────────────────────────────────────┐
│  Browser    │  Support   │  Notes           │
├─────────────┼────────────┼──────────────────┤
│  Chrome     │  ✓ 6+      │  Full support    │
│  Firefox    │  ✓ 6+      │  Full support    │
│  Safari     │  ✓ 5+      │  Full support    │
│  Edge       │  ✓ 79+     │  Full support    │
│  IE 11      │  ✗         │  Polyfill လိုသည် │
│  Opera      │  ✓ 11+     │  Full support    │
└─────────────┴────────────┴──────────────────┘
```

---

## ၁၅.၂ Use Cases: Live Feeds, Dashboards, Notifications

SSE သည် သတ်မှတ်ထားသော scenarios များတွင် အထူးကောင်းမွန်သည်။

### Real-time Dashboard

```javascript
// Stock market dashboard
class StockDashboard {
    constructor() {
        this.es = new EventSource('/api/stocks/stream');
        this.stockData = new Map();
        this.setupEventListeners();
    }

    setupEventListeners() {
        // price update receive လုပ်သည်
        this.es.addEventListener('priceUpdate', (event) => {
            const { symbol, price, change } = JSON.parse(event.data);
            this.updateStockCard(symbol, price, change);
        });

        // market alert receive လုပ်သည်
        this.es.addEventListener('alert', (event) => {
            const alert = JSON.parse(event.data);
            this.showNotification(alert);
        });

        // trading volume update
        this.es.addEventListener('volume', (event) => {
            const volumeData = JSON.parse(event.data);
            this.updateVolumeChart(volumeData);
        });
    }

    updateStockCard(symbol, price, change) {
        const card = document.querySelector(`[data-symbol="${symbol}"]`);
        if (card) {
            card.querySelector('.price').textContent = `$${price}`;
            // price တက်/ကျ color ပြောင်းသည်
            const changeEl = card.querySelector('.change');
            changeEl.textContent = `${change > 0 ? '+' : ''}${change}%`;
            changeEl.className = `change ${change > 0 ? 'up' : 'down'}`;
        }
    }
}
```

### Progress Notification Stream

```javascript
// Server-side: long-running task progress stream
app.post('/api/export', async (req, res) => {
    const jobId = generateJobId();
    
    // job ကို background တွင် start လုပ်သည်
    processExportJob(jobId, req.body);
    
    // job ID ကို ချက်ချင်း return လုပ်သည်
    res.json({ jobId });
});

app.get('/api/export/:jobId/progress', (req, res) => {
    const { jobId } = req.params;
    
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');

    // progress updates ကို stream လုပ်သည်
    const onProgress = ({ percent, stage, message }) => {
        res.write(`data: ${JSON.stringify({
            percent,
            stage,
            message,
            timestamp: Date.now()
        })}\n\n`);

        // 100% ရောက်သောအခါ stream ပိတ်သည်
        if (percent >= 100) {
            res.write(`event: complete\ndata: {"jobId": "${jobId}"}\n\n`);
            res.end();
        }
    };

    jobEventEmitter.on(`progress:${jobId}`, onProgress);

    req.on('close', () => {
        jobEventEmitter.off(`progress:${jobId}`, onProgress);
    });
});
```

---

## ၁၅.၃ SSE vs WebSocket Decision Guide

မည်သည့် technology ကို ရွေးချယ်ရမည်ဆိုသည် question သည် use case ပေါ်တွင် မူတည်သည်။

```
Use Case ကို အကဲဖြတ်ပါ:
         │
         ▼
┌─────────────────────────┐
│ Client → Server data    │
│ ပို့ဆောင်ရန် လိုသလား?    │
└─────────────────────────┘
         │
    ┌────┴────┐
    │   YES   │   NO
    │         │
    ▼         ▼
WebSocket    ┌──────────────────────┐
             │ High frequency       │
             │ updates လိုသလား?     │
             │ (>1 msg/sec)         │
             └──────────────────────┘
                      │
               ┌──────┴──────┐
               │     YES     │   NO
               │             │
               ▼             ▼
           WebSocket        SSE ✓
```

### Comparison Table

```
┌──────────────────────────┬────────────────┬────────────────┐
│ Criteria                 │     SSE        │   WebSocket    │
├──────────────────────────┼────────────────┼────────────────┤
│ Direction                │ Server→Client  │ Bidirectional  │
│ Protocol                 │ HTTP/1.1       │ WS             │
│ Auto-reconnect           │ Built-in       │ Manual         │
│ Proxy/Firewall friendly  │ Yes            │ Sometimes      │
│ Load balancing           │ Easy           │ Sticky needed  │
│ HTTP/2 multiplexing      │ Yes            │ No             │
│ Binary data              │ No (text only) │ Yes            │
│ Implementation           │ Simple         │ Complex        │
│ Browser polyfill         │ Easy           │ Not needed     │
└──────────────────────────┴────────────────┴────────────────┘
```

---

## ၁၅.၄ Chunked HTTP Responses

Chunked Transfer Encoding သည် server မှ data ကို pieces ဖြင့် ပေးပို့နိုင်သည်။

```javascript
// Express တွင် chunked response
app.get('/api/large-data', async (req, res) => {
    res.setHeader('Transfer-Encoding', 'chunked');
    res.setHeader('Content-Type', 'application/json');

    // large dataset ကို chunks ဖြင့် ပေးပို့သည်
    const cursor = await database.query('SELECT * FROM large_table');
    
    res.write('[\n');
    let firstRow = true;
    
    cursor.on('data', (row) => {
        if (!firstRow) res.write(',\n');
        // တစ်ကြောင်းချင်းစီ chunk ဖြင့် ပေးပို့သည်
        res.write(JSON.stringify(row));
        firstRow = false;
    });

    cursor.on('end', () => {
        res.write('\n]');
        res.end();
    });

    cursor.on('error', (err) => {
        res.end();
    });
});
```

### HTTP/2 Server Push vs Chunked

```
HTTP/1.1 Chunked:
Client ──── GET /large-file ────► Server
            ◄── chunk 1 ─────────
            ◄── chunk 2 ─────────
            ◄── chunk 3 ─────────
            ◄── chunk N ─────────

HTTP/2 with SSE:
Client ──── GET /stream ─────────► Server
            ◄══ frame 1 ══════════
            ◄══ frame 2 ══════════  ← multiplexed over single connection
            ◄══ frame 3 ══════════
```

---

## ၁၅.၅ Streaming with gRPC Server Streams

gRPC သည် protocol buffers ဖြင့် efficient binary streaming ပေးဆောင်သည်။

### gRPC Service Definition

```protobuf
// stock_service.proto
syntax = "proto3";

package stock;

service StockService {
    // Server streaming RPC — server မှ stream ပေးပို့သည်
    rpc SubscribePrices (PriceRequest) returns (stream PriceUpdate);
    
    // Bidirectional streaming
    rpc TradeStream (stream TradeOrder) returns (stream TradeConfirmation);
}

message PriceRequest {
    repeated string symbols = 1;  // subscribe လုပ်မည့် symbols
}

message PriceUpdate {
    string symbol = 1;
    double price = 2;
    double change_percent = 3;
    int64 timestamp = 4;
}
```

### gRPC Server Implementation (Node.js)

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

// proto file load လုပ်သည်
const packageDef = protoLoader.loadSync('stock_service.proto');
const stockProto = grpc.loadPackageDefinition(packageDef).stock;

const stockService = {
    // server streaming ကို implement လုပ်သည်
    subscribePrices: (call) => {
        const { symbols } = call.request;
        console.log(`Subscribing to: ${symbols.join(', ')}`);

        // price update interval ဖန်တီးသည်
        const interval = setInterval(() => {
            symbols.forEach((symbol) => {
                // simulate price update
                const update = {
                    symbol,
                    price: Math.random() * 1000 + 100,
                    changePercent: (Math.random() - 0.5) * 10,
                    timestamp: Date.now()
                };

                // stream ဖြင့် data ပေးပို့သည်
                call.write(update);
            });
        }, 1000);

        // client ချိတ်ဆက်မှု ဖြတ်သောအခါ cleanup
        call.on('cancelled', () => {
            clearInterval(interval);
        });
    }
};

// server စတင်သည်
const server = new grpc.Server();
server.addService(stockProto.StockService.service, stockService);
server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => {
    server.start();
    console.log('gRPC server running on port 50051');
});
```

### gRPC Client (Streaming Consumer)

```javascript
const client = new stockProto.StockService(
    'localhost:50051',
    grpc.credentials.createInsecure()
);

// server streaming call ဖန်တီးသည်
const stream = client.subscribePrices({ symbols: ['AAPL', 'GOOGL', 'MSFT'] });

// stream မှ data receive လုပ်သည်
stream.on('data', (priceUpdate) => {
    console.log(`${priceUpdate.symbol}: $${priceUpdate.price.toFixed(2)}`);
    updatePriceDisplay(priceUpdate);
});

stream.on('end', () => {
    console.log('Stream ပြီးဆုံးသွားသည်');
});

stream.on('error', (error) => {
    console.error('Stream error:', error.message);
    // reconnect logic ထည့်နိုင်သည်
});
```

### SSE vs gRPC Streaming Comparison

```
┌──────────────────────┬──────────────────┬────────────────────┐
│ Feature              │    SSE           │  gRPC Streaming    │
├──────────────────────┼──────────────────┼────────────────────┤
│ Protocol             │ HTTP/1.1 or 2    │ HTTP/2             │
│ Data Format          │ Text only        │ Binary (Protobuf)  │
│ Browser Support      │ Native           │ grpc-web needed    │
│ Performance          │ Good             │ Excellent          │
│ Type Safety          │ Manual           │ Generated code     │
│ Service-to-Service   │ Possible         │ Ideal              │
│ Mobile clients       │ Good             │ Better             │
│ Debugging            │ Easy (text)      │ Harder (binary)    │
└──────────────────────┴──────────────────┴────────────────────┘
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

- **SSE Protocol:** `text/event-stream` MIME type ဖြင့် server မှ client သို့ persistent one-way stream ဖွင့်ပြီး `data:`, `event:`, `id:`, `retry:` fields များ အသုံးပြုသည်
- **Auto-Reconnect:** SSE ၏ built-in reconnection feature သည် `Last-Event-ID` header ဖြင့် missed messages များကို ရယူနိုင်သောကြောင့် WebSocket ထက် resilient ဖြစ်သည်
- **Use Case Fit:** Notifications, dashboards, live feeds ကဲ့သို့ server→client only scenarios တွင် SSE သည် WebSocket ထက် implement လုပ်ရ ပိုလွယ်ကူသည်
- **HTTP/2 Advantage:** HTTP/2 ဖြင့် SSE ကို multiplex လုပ်ခြင်းသည် single connection ပေါ်တွင် multiple streams ဖြစ်နိုင်သောကြောင့် browser connection limits ကို ကျော်လွှားနိုင်သည်
- **gRPC Streaming:** Service-to-service communication တွင် gRPC server streaming သည် binary efficiency နှင့် type safety ပေးဆောင်သောကြောင့် high-performance scenarios တွင် ပိုသင့်တော်သည်
- **Decision Guide:** Client မှ data မပေးပို့ဘဲ server မှ stream ကိုသာ လိုအပ်ပါက SSE ကို ရွေးပါ; bidirectional communication လိုပါက WebSocket ကို ရွေးပါ; internal microservices streaming တွင် gRPC ကို ထည့်သွင်းစဉ်းစားပါ
