# အခန်း ၁၄: WebSocket နှင့် Persistent Connections

## နိဒါန်း

ခေတ်မီ web application များတွင် real-time communication သည် မရှိမဖြစ် လိုအပ်သောအရာဖြစ်လာသည်။ Chat application များ၊ live dashboard များ၊ online gaming systems များ၊ collaborative editing tools များ — ဤ application များအားလုံးတွင် server နှင့် client ကြား အချိန်နှင့်တပြေးညီ data exchange လုပ်ဆောင်ရန် လိုအပ်သည်။ ဤအခန်းတွင် WebSocket နှင့် persistent connection နည်းပညာများကို နက်နဲစွာ လေ့လာမည်ဖြစ်ပြီး၊ millions of connections ကို scale ချနိုင်သော architecture များကိုလည်း ဆွေးနွေးမည်ဖြစ်သည်။

---

## ၁၄.၁ HTTP Long Polling vs SSE vs WebSocket

Real-time communication ကို implement လုပ်ရာတွင် approach သုံးမျိုးရှိသည်။ ၎င်းတို့၏ ကွာခြားချက်ကို နားလည်ခြင်းသည် မှန်ကန်သော tool ရွေးချယ်ရန် အရေးကြီးသည်။

### HTTP Long Polling

Long polling သည် HTTP ၏ request-response model ကို အသုံးပြု၍ real-time ကို simulate လုပ်သည်။

```
Client                          Server
  |                               |
  |------- HTTP Request -------->|  ← client က request ပို့သည်
  |                               |  ← server က data မရသေးဘဲ ထားသည် (hold)
  |                               |  ← data ရှိလာမှ response ပြန်ပို့သည်
  |<------ HTTP Response --------|
  |                               |
  |------- HTTP Request -------->|  ← ချက်ချင်း request အသစ် ပို့သည်
  |                               |
```

**ဆိုးကျိုးများ:** တစ်ခုချင်းစီ connection ဖွင့်ပိတ် လုပ်ရသောကြောင့် overhead များသည်။ Latency မြင့်သည်။

### Server-Sent Events (SSE)

SSE သည် server မှ client သို့ one-way stream ဖြင့် data ပေးပို့သည်။

```
Client                          Server
  |                               |
  |------- HTTP GET Request ----->|
  |<====== Persistent Stream =====|  ← server က data stream ဖွင့်သည်
  |<====== Event 1 ===============|
  |<====== Event 2 ===============|
  |<====== Event 3 ===============|
  |          (ဆက်လက် stream)      |
```

### WebSocket

WebSocket သည် full-duplex, persistent connection တည်ဆောက်သည်။

```
Client                          Server
  |                               |
  |--- HTTP Upgrade Request ----->|  ← handshake စတင်
  |<-- 101 Switching Protocols ---|  ← server က လက်ခံ
  |                               |
  |<======= Bi-directional ======>|  ← both ways data ပို့နိုင်သည်
  |======== Message 1 ==========>|
  |<======= Message 2 ============|
  |======== Message 3 ==========>|
```

**နှိုင်းယှဉ်ချက်ဇယား:**

```
┌─────────────────┬──────────────┬──────────────┬──────────────┐
│ Feature         │ Long Polling │     SSE      │  WebSocket   │
├─────────────────┼──────────────┼──────────────┼──────────────┤
│ Direction       │  Two-way     │  One-way     │  Two-way     │
│ Protocol        │  HTTP        │  HTTP        │  WS/WSS      │
│ Browser Support │  All         │  Most        │  All modern  │
│ Complexity      │  Low         │  Low         │  Medium      │
│ Latency         │  High        │  Low         │  Lowest      │
│ Overhead        │  High        │  Low         │  Lowest      │
└─────────────────┴──────────────┴──────────────┴──────────────┘
```

---

## ၁၄.၂ WebSocket Handshake နှင့် Protocol

WebSocket connection သည် standard HTTP request ဖြင့် စတင်ပြီး protocol upgrade ဖြင့် WebSocket protocol သို့ ပြောင်းလဲသည်။

### Handshake Process

**Client က ပေးပို့သော HTTP Request:**

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

**Server ၏ Response:**

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### WebSocket Frame Structure

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - -+
```

### Node.js တွင် Basic WebSocket Server

```javascript
const WebSocket = require('ws');

// WebSocket server ဖန်တီးသည်
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws, request) => {
    // client IP address ရယူသည်
    const clientIP = request.socket.remoteAddress;
    console.log(`Client ချိတ်ဆက်လာသည်: ${clientIP}`);

    // message လက်ခံသည်
    ws.on('message', (message) => {
        console.log(`လက်ခံရသည်: ${message}`);
        
        // client အားလုံးထံ broadcast လုပ်သည်
        wss.clients.forEach((client) => {
            if (client.readyState === WebSocket.OPEN) {
                // JSON format ဖြင့် message ပေးပို့သည်
                client.send(JSON.stringify({
                    type: 'broadcast',
                    data: message.toString(),
                    timestamp: Date.now()
                }));
            }
        });
    });

    // connection ပိတ်သွားသောအခါ
    ws.on('close', (code, reason) => {
        console.log(`Client ချိတ်ဆက်မှု ဖြတ်သွားသည်: ${code}`);
    });

    // error handling
    ws.on('error', (error) => {
        console.error(`WebSocket error: ${error.message}`);
    });
});
```

---

## ၁၄.၃ Connection Management at Scale (Millions of Connections)

Production environment တွင် millions of concurrent WebSocket connections ကို handle လုပ်ရခြင်းသည် engineering challenge ကြီးတစ်ခုဖြစ်သည်။

### Operating System Level Optimization

```bash
# Linux kernel parameters များ optimize လုပ်သည်
# /etc/sysctl.conf တွင် ထည့်ပါ

# file descriptors အများဆုံး ပမာဏ
fs.file-max = 2000000

# TCP connection တစ်ခုချင်းစီ memory သုံးသောပမာဏ လျှော့ချသည်
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# TIME_WAIT connections များကို reuse လုပ်ခွင့်ပေးသည်
net.ipv4.tcp_tw_reuse = 1
```

### Connection State Management

```javascript
class ConnectionManager {
    constructor() {
        // userId → WebSocket connection map
        this.connections = new Map();
        // connection metadata သိမ်းသည်
        this.metadata = new Map();
    }

    // connection အသစ် ထည့်သည်
    addConnection(userId, ws) {
        this.connections.set(userId, ws);
        this.metadata.set(userId, {
            connectedAt: Date.now(),
            lastActivity: Date.now(),
            messageCount: 0
        });
    }

    // user ထံ message ပေးပို့သည်
    sendToUser(userId, message) {
        const ws = this.connections.get(userId);
        if (ws && ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify(message));
            // activity time update လုပ်သည်
            const meta = this.metadata.get(userId);
            if (meta) {
                meta.lastActivity = Date.now();
                meta.messageCount++;
            }
            return true;
        }
        return false;
    }

    // inactive connections များ ရှင်းလင်းသည်
    cleanupInactive(timeoutMs = 30000) {
        const now = Date.now();
        for (const [userId, meta] of this.metadata.entries()) {
            if (now - meta.lastActivity > timeoutMs) {
                this.removeConnection(userId);
            }
        }
    }

    removeConnection(userId) {
        const ws = this.connections.get(userId);
        if (ws) {
            ws.close();
        }
        this.connections.delete(userId);
        this.metadata.delete(userId);
    }
}
```

---

## ၁၄.၄ Presence Detection နှင့် Heartbeats

User များ online/offline status ကို detect လုပ်ရန် heartbeat mechanism လိုအပ်သည်။

### Ping/Pong Heartbeat Implementation

```javascript
class HeartbeatManager {
    constructor(ws, userId) {
        this.ws = ws;
        this.userId = userId;
        this.isAlive = true;
        this.interval = null;
    }

    start(intervalMs = 30000) {
        // WebSocket ၏ built-in ping/pong mechanism သုံးသည်
        this.ws.on('pong', () => {
            // pong လက်ခံရရှိသည် — connection အသက်ရှင်နေသည်
            this.isAlive = true;
        });

        this.interval = setInterval(() => {
            if (!this.isAlive) {
                // heartbeat မရ — connection သေသည်
                console.log(`User ${this.userId} connection timeout`);
                this.ws.terminate();
                return;
            }

            this.isAlive = false;
            // ping ပေးပို့သည်
            this.ws.ping();
        }, intervalMs);
    }

    stop() {
        if (this.interval) {
            clearInterval(this.interval);
        }
    }
}

// Presence Service — user online/offline status စီမံသည်
class PresenceService {
    constructor(redisClient) {
        this.redis = redisClient;
        // online user များ key prefix
        this.PRESENCE_PREFIX = 'presence:';
        // TTL: 60 seconds
        this.TTL = 60;
    }

    async setOnline(userId) {
        // Redis တွင် online status သိမ်းသည်
        await this.redis.setex(
            `${this.PRESENCE_PREFIX}${userId}`,
            this.TTL,
            JSON.stringify({ status: 'online', timestamp: Date.now() })
        );
    }

    async setOffline(userId) {
        await this.redis.del(`${this.PRESENCE_PREFIX}${userId}`);
    }

    async isOnline(userId) {
        const data = await this.redis.get(`${this.PRESENCE_PREFIX}${userId}`);
        return data !== null;
    }

    async refreshPresence(userId) {
        // TTL ကို refresh လုပ်သည် — user အသက်ရှင်နေကြောင်း သက်သေပြသည်
        await this.redis.expire(
            `${this.PRESENCE_PREFIX}${userId}`,
            this.TTL
        );
    }
}
```

---

## ၁၄.၅ Horizontal Scaling WebSocket Servers

WebSocket server များကို horizontal scale လုပ်သောအခါ challenge ကြီးတစ်ခုဖြစ်သည် — connection တစ်ခုသည် specific server instance တစ်ခုနှင့်သာ ချိတ်ဆက်ထားသောကြောင့်ဖြစ်သည်။

### Sticky Sessions

```
                    ┌─────────────────────────────┐
                    │       Load Balancer          │
                    │   (IP Hash / Cookie Based)   │
                    └──────┬──────┬──────┬─────────┘
                           │      │      │
               ┌───────────▼──┐ ┌─▼─────▼──┐ ┌──▼───────┐
               │  WS Server 1 │ │WS Server 2│ │WS Server3│
               │ (User A,B)   │ │(User C,D) │ │(User E,F)│
               └──────────────┘ └───────────┘ └──────────┘
```

### Redis Pub/Sub ဖြင့် Cross-Server Messaging

```javascript
const Redis = require('ioredis');

class ScalableWebSocketServer {
    constructor() {
        // publish လုပ်ရန် Redis client
        this.publisher = new Redis({ host: 'redis-host', port: 6379 });
        // subscribe လုပ်ရန် Redis client (separate connection လိုအပ်သည်)
        this.subscriber = new Redis({ host: 'redis-host', port: 6379 });
        
        // local connections (ဤ server instance ပေါ်တွင်ရှိသော)
        this.localConnections = new Map();
        
        // Redis channel subscribe လုပ်သည်
        this.subscriber.subscribe('ws-messages', (err) => {
            if (err) console.error('Subscribe error:', err);
        });

        // Redis မှ message လက်ခံသောအခါ
        this.subscriber.on('message', (channel, message) => {
            const data = JSON.parse(message);
            this.deliverLocally(data);
        });
    }

    // message ကို Redis ဖြင့် broadcast လုပ်သည်
    async broadcastMessage(targetUserId, message) {
        await this.publisher.publish('ws-messages', JSON.stringify({
            targetUserId,
            message,
            serverId: process.env.SERVER_ID
        }));
    }

    // ဤ server တွင် ရှိနေသော connection ထံ deliver လုပ်သည်
    deliverLocally(data) {
        const { targetUserId, message } = data;
        const ws = this.localConnections.get(targetUserId);
        if (ws && ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify(message));
        }
    }
}
```

### Architecture Diagram

```
Client A ──────► WS Server 1 ──────► Redis Pub/Sub
                                            │
Client B ──────► WS Server 2 ──────►  Channel: ws-messages
                                            │
Client C ──────► WS Server 3 ──────────────┘
                     │
              Subscribe: WS Server 1, 2, 3 အားလုံး
              → message လက်ခံသောအခါ local connections စစ်ဆေးသည်
```

---

## ၁၄.၆ Reconnection Strategy နှင့် Message Ordering

Network ချိတ်ဆက်မှု ဖြတ်တောက်သွားသောအခါ client သည် reconnect လုပ်ရမည်ဖြစ်ပြီး missed messages များကိုလည်း ရယူနိုင်ရမည်။

### Exponential Backoff Reconnection

```javascript
class WebSocketClient {
    constructor(url) {
        this.url = url;
        this.ws = null;
        // reconnect ကြိမ်ရေ
        this.retryCount = 0;
        // အများဆုံး retry ကြိမ်ရေ
        this.maxRetries = 10;
        // base delay: 1 second
        this.baseDelay = 1000;
        // message sequence number ကို track လုပ်သည်
        this.lastSequenceNumber = 0;
    }

    connect() {
        this.ws = new WebSocket(this.url);

        this.ws.onopen = () => {
            console.log('Connected successfully');
            // reconnect ဖြစ်ပါက missed messages တောင်းသည်
            if (this.retryCount > 0) {
                this.requestMissedMessages();
            }
            this.retryCount = 0;
        };

        this.ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            // sequence number စစ်ဆေးသည် — ordering အတွက်
            if (data.seq > this.lastSequenceNumber + 1) {
                // messages များ missing ဖြစ်နေသည်
                this.requestMissedMessages(this.lastSequenceNumber + 1, data.seq - 1);
            }
            this.lastSequenceNumber = data.seq;
            this.handleMessage(data);
        };

        this.ws.onclose = () => {
            this.scheduleReconnect();
        };
    }

    scheduleReconnect() {
        if (this.retryCount >= this.maxRetries) {
            console.error('Maximum retry attempts reached');
            return;
        }

        // Exponential backoff with jitter ဖြင့် delay တွက်ချက်သည်
        const delay = Math.min(
            this.baseDelay * Math.pow(2, this.retryCount) + 
            Math.random() * 1000,  // jitter ထည့်သည်
            30000  // maximum 30 seconds
        );

        console.log(`${delay}ms နောက်မှ reconnect လုပ်မည်...`);
        setTimeout(() => {
            this.retryCount++;
            this.connect();
        }, delay);
    }

    requestMissedMessages(from, to) {
        // server ထံ missed messages တောင်းဆိုသည်
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify({
                type: 'request_missed',
                fromSeq: from || this.lastSequenceNumber,
                toSeq: to
            }));
        }
    }
}
```

### Message Queue with Ordering Guarantee

```javascript
class MessageOrderingBuffer {
    constructor() {
        // ဆိုင်းငံ့ထားသော messages
        this.pendingMessages = new Map();
        // လက်ရှိ expected sequence number
        this.expectedSeq = 1;
    }

    addMessage(seq, message) {
        if (seq === this.expectedSeq) {
            // order မှန်ကန်နေသည် — ချက်ချင်း process လုပ်သည်
            this.processMessage(message);
            this.expectedSeq++;
            // buffer တွင် ဆက်တိုက်ရှိနေသော messages ထုတ်ယူသည်
            this.flushBuffer();
        } else if (seq > this.expectedSeq) {
            // future message — buffer တွင် သိမ်းထားသည်
            this.pendingMessages.set(seq, message);
        }
        // seq < expectedSeq ဆိုသည် duplicate — ignore လုပ်သည်
    }

    flushBuffer() {
        while (this.pendingMessages.has(this.expectedSeq)) {
            const message = this.pendingMessages.get(this.expectedSeq);
            this.pendingMessages.delete(this.expectedSeq);
            this.processMessage(message);
            this.expectedSeq++;
        }
    }

    processMessage(message) {
        // application logic ကို call လုပ်သည်
        console.log('Message processing:', message);
    }
}
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

- **Protocol ရွေးချယ်မှု:** Long Polling သည် simple cases အတွက် သင့်တော်သည်၊ SSE သည် server-to-client streaming အတွက် ကောင်းသည်၊ WebSocket သည် full-duplex real-time communication အတွက် အကောင်းဆုံးဖြစ်သည်
- **Handshake:** WebSocket connection သည် HTTP Upgrade request ဖြင့် စတင်ပြီး 101 Switching Protocols response ရရှိမှ WebSocket protocol သို့ ကူးပြောင်းသည်
- **Scale:** Millions of connections ကို handle လုပ်ရန် OS-level tuning၊ sticky sessions၊ နှင့် Redis Pub/Sub pattern များ အသုံးပြုရသည်
- **Heartbeats:** Ping/Pong mechanism ဖြင့် dead connections ကို detect လုပ်ပြီး presence detection ကို Redis TTL ဖြင့် implement လုပ်နိုင်သည်
- **Reconnection:** Exponential backoff with jitter strategy ဖြင့် reconnect လုပ်ပြီး sequence numbers ဖြင့် missed/duplicate messages ကို detect လုပ်နိုင်သည်; end-to-end ordering guarantee လိုပါက server-side replay logic ထပ်လိုအပ်သည်
- **Horizontal Scaling:** Redis Pub/Sub ဖြင့် WebSocket servers အကြား message routing လုပ်ဆောင်၍ horizontal scaling ကို enable လုပ်နိုင်သည်
