# အခန်း ၂၀: Service Discovery နှင့် Load Balancing

## နိဒါန်း

Microservices architecture တွင် services များသည် dynamically scale up/down ဖြစ်နေပြီး IP addresses များလည်း အမြဲပြောင်းလဲနေပါသည်။ Container orchestration (Kubernetes) ပတ်ဝန်းကျင်တွင် pod တစ်ခု restart ဖြစ်တိုင်း IP address ပြောင်းသွားသည်။ ထို့ကြောင့် Service A သည် Service B ကို call လုပ်ရန် B ၏ location ကို dynamically ရှာဖွေနိုင်ရမည်။ ဤ problem ကို ဖြေရှင်းပေးသည်မှာ **Service Discovery** ဖြစ်ပါသည်။ ထို့နောက် traffic ကို multiple instances သို့ ညီညာစွာ ဖြန့်ဝေပေးသည်မှာ **Load Balancing** ဖြစ်သည်။ ဤအခန်းတွင် ဤ mechanisms နှစ်ခုလုံးကို အသေးစိတ် လေ့လာပါမည်။

---

## ၂၀.၁ Client-Side vs Server-Side Discovery

Service discovery တွင် ချဉ်းကပ်နည်း အဓိက ၂ မျိုး ရှိပါသည်။

### Server-Side Discovery

Client သည် load balancer/router ကိုသာ request ပို့ပြီး load balancer က service instance ကို ရွေးချယ်ပေးသည်။

```
                    +------------------+
                    |  Load Balancer   |
                    |  (Nginx/AWS ALB) |
   +--------+      +--------+---------+      +-----------+
   | Client  | ---> |        |         | ---> | Service A |
   +--------+      |        |         |      | Instance 1|
                    |        |         |      +-----------+
                    |        |         | ---> +-----------+
                    |        |         |      | Service A |
                    +--------+---------+      | Instance 2|
                                              +-----------+
```

### Client-Side Discovery

Client သည် service registry ကို query လုပ်ပြီး မိမိကိုယ်တိုင် service instance ကို ရွေးချယ်သည်။

```
   +--------+      +------------------+
   | Client  | ---> | Service Registry |   (1) Registry query လုပ်ခြင်း
   +--------+      | (Consul/Eureka)  |
       |            +------------------+
       |
       |  (2) တိုက်ရိုက် call လုပ်ခြင်း
       |            +-----------+
       +----------> | Service A |
                    | Instance 1|
                    +-----------+
```

```python
# Client-side discovery ဥပမာ - Service Registry မှ instance ရွေးချယ်ခြင်း
import consul
import random

class ServiceDiscovery:
    def __init__(self):
        # Consul client ချိတ်ဆက်ခြင်း
        self.consul_client = consul.Consul(host='localhost', port=8500)

    def get_service_url(self, service_name):
        """Service name ဖြင့် healthy instance တစ်ခု၏ URL ကို ပြန်ပေးခြင်း"""
        # Consul မှ healthy services များကို query လုပ်ခြင်း
        _, services = self.consul_client.health.service(
            service_name,
            passing=True  # Healthy instances ကိုသာ ယူခြင်း
        )

        if not services:
            raise Exception(f"Service '{service_name}' not found")

        # Random load balancing - instance တစ်ခုကို ကျပန်း ရွေးချယ်ခြင်း
        instance = random.choice(services)
        address = instance['Service']['Address']
        port = instance['Service']['Port']
        return f"http://{address}:{port}"

# အသုံးပြုနည်း
discovery = ServiceDiscovery()
order_service_url = discovery.get_service_url("order-service")
print(f"Order Service URL: {order_service_url}")
```

### နှိုင်းယှဉ်ခြင်း

| Feature | Client-Side | Server-Side |
|---------|-------------|-------------|
| Latency | နိမ့်သည် (direct call) | အနည်းငယ် မြင့်သည် (extra hop) |
| Client Complexity | မြင့်သည် | နိမ့်သည် |
| Language Independent | မဟုတ် | ဟုတ်သည် |
| Single Point of Failure | မရှိ | Load balancer |

---

## ၂၀.၂ Service Registry (Consul, Eureka, etcd)

Service Registry သည် service instances များ၏ network locations များကို သိမ်းဆည်းထားသော database ဖြစ်ပါသည်။

```
+--------------------------------------------------------------+
|                    Service Registry                           |
+--------------------------------------------------------------+
| Service Name    | Instance ID | Address        | Health      |
|-----------------|-------------|----------------|-------------|
| order-service   | order-1     | 10.0.1.5:8080  | healthy     |
| order-service   | order-2     | 10.0.1.6:8080  | healthy     |
| order-service   | order-3     | 10.0.1.7:8080  | unhealthy   |
| payment-service | payment-1   | 10.0.2.3:8081  | healthy     |
| payment-service | payment-2   | 10.0.2.4:8081  | healthy     |
+--------------------------------------------------------------+
```

### Consul

HashiCorp မှ ဖန်တီးထားပြီး service discovery, health checking, KV store နှင့် multi-datacenter support ပါဝင်သည်။

```python
# Consul ဖြင့် service registration
import consul

c = consul.Consul()

# Service register လုပ်ခြင်း - health check ပါ ပူးတွဲခြင်း
c.agent.service.register(
    name="order-service",
    service_id="order-service-1",          # Unique instance ID
    address="10.0.1.5",
    port=8080,
    tags=["v2", "production"],             # Tags - version, environment
    check=consul.Check.http(               # HTTP health check
        url="http://10.0.1.5:8080/health",
        interval="10s",                     # ၁၀ စက္ကန့်တိုင်း စစ်ဆေးခြင်း
        timeout="5s",                       # Timeout ၅ စက္ကန့်
        deregister="30s"                    # Unhealthy ဖြစ်လျှင် ၃၀ စက္ကန့်နောက် ဖယ်ရှားခြင်း
    )
)
print("Service registered with Consul")    # Consul တွင် register ပြီးကြောင်း
```

### Eureka (Netflix OSS)

Java/Spring ecosystem တွင် popular ဖြစ်ပြီး Spring Cloud နှင့် လွယ်ကူစွာ integrate လုပ်နိုင်သည်။

```java
// Spring Boot application.yml - Eureka client configuration
// Eureka server သို့ register လုပ်ခြင်းနှင့် service ရှာဖွေခြင်း
/*
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
    fetch-registry: true          # Registry data ကို local cache လုပ်ခြင်း
    registry-fetch-interval: 5    # ၅ စက္ကန့်တိုင်း refresh လုပ်ခြင်း
  instance:
    prefer-ip-address: true       # Hostname အစား IP address သုံးခြင်း
    lease-renewal-interval: 10    # Heartbeat interval (စက္ကန့်)
    lease-expiration-duration: 30 # Heartbeat မရှိလျှင် expire ဖြစ်ခြင်း
*/
```

### etcd

Kubernetes ၏ backing store အဖြစ် အသုံးပြုသော distributed key-value store ဖြစ်ပြီး strong consistency (Raft consensus) ပေးစွမ်းသည်။

---

## ၂၀.၃ Load Balancing Strategies

Traffic ကို service instances များသို့ ဖြန့်ဝေရာတွင် algorithm အမျိုးမျိုး ရှိပါသည်။

### Round Robin

Request များကို instances များသို့ အလှည့်ကျ ဖြန့်ဝေခြင်း ဖြစ်ပါသည်။ ရိုးရှင်းသော်လည်း instance ၏ load ကို ထည့်မစဉ်းစားပါ။

```python
# Round Robin load balancer implementation
class RoundRobinBalancer:
    def __init__(self, instances):
        self.instances = instances  # Service instance စာရင်း
        self.current = 0           # လက်ရှိ index

    def get_next(self):
        """နောက်တစ်ခု instance ကို ပြန်ပေးခြင်း"""
        instance = self.instances[self.current % len(self.instances)]
        self.current += 1
        return instance

# ဥပမာ - instances ၃ ခုသို့ အလှည့်ကျ ဖြန့်ဝေခြင်း
balancer = RoundRobinBalancer(["10.0.1.1", "10.0.1.2", "10.0.1.3"])
for i in range(6):
    print(f"Request {i+1} -> {balancer.get_next()}")
# Output: 10.0.1.1, 10.0.1.2, 10.0.1.3, 10.0.1.1, 10.0.1.2, 10.0.1.3
```

### Least Connections

Active connection အနည်းဆုံးသော instance သို့ request ပို့ခြင်း ဖြစ်သည်။ Long-lived connections (WebSocket) တွင် ပိုသင့်လျော်သည်။

```python
# Least Connections load balancer
import heapq

class LeastConnectionsBalancer:
    def __init__(self, instances):
        # Min-heap - (active_connections, instance_address)
        self.heap = [(0, addr) for addr in instances]
        heapq.heapify(self.heap)

    def acquire(self):
        """Connection အနည်းဆုံး instance ကို ရယူခြင်း"""
        connections, addr = heapq.heappop(self.heap)
        heapq.heappush(self.heap, (connections + 1, addr))
        return addr

    def release(self, addr):
        """Connection ပြန်လွှတ်ခြင်း"""
        self.heap = [(c - 1 if a == addr else c, a) for c, a in self.heap]
        heapq.heapify(self.heap)
```

### Consistent Hashing

Request ကို hash value အလိုက် instance သို့ route လုပ်ခြင်း ဖြစ်သည်။ Cache affinity လိုအပ်သော use case (session stickiness) တွင် အသုံးဝင်သည်။

```
            Hash Ring (0 to 2^32)
           +----+
          /  N1  \         N = Node (instance)
         /        \        R = Request
    +---+    +---+ +---+
    |N3 |    | R1| |N2 |   R1 -> N2 (clockwise ဖြင့် ရှာခြင်း)
    +---+    +---+ +---+
         \        /
          \ +---+/
           \|R2 |/
            +---+              R2 -> N3
```

---

## ၂၀.၄ DNS-Based Discovery in Kubernetes

Kubernetes တွင် built-in DNS-based service discovery ပါဝင်ပြီး CoreDNS (သို့) kube-dns ဖြင့် service names များကို cluster IP addresses များသို့ resolve လုပ်ပေးသည်။

```yaml
# Kubernetes Service definition
# ClusterIP service - cluster အတွင်း discovery အတွက်
apiVersion: v1
kind: Service
metadata:
  name: order-service           # DNS name ဖြစ်လာမည်
  namespace: production
spec:
  selector:
    app: order-service          # ဤ label ရှိသော pods များကို target လုပ်ခြင်း
  ports:
    - port: 80                  # Service port
      targetPort: 8080          # Pod port
  type: ClusterIP               # Cluster internal - external access မရှိ
```

```
DNS Resolution in Kubernetes:

order-service                              -> ClusterIP (same namespace)
order-service.production                   -> ClusterIP (cross namespace)
order-service.production.svc.cluster.local -> FQDN (fully qualified)

Headless Service (clusterIP: None):
order-service.production.svc.cluster.local -> Pod IPs directly
  -> 10.244.1.5
  -> 10.244.2.8
  -> 10.244.3.3
```

```python
# Kubernetes environment တွင် service discovery
import requests

# Kubernetes DNS ဖြင့် service ခေါ်ခြင်း - IP address မလိုဘဲ name ဖြင့် ခေါ်နိုင်ခြင်း
response = requests.get("http://order-service.production.svc.cluster.local/api/orders")

# Same namespace ဆိုလျှင် service name ဖြင့်သာ လုံလောက်ခြင်း
response = requests.get("http://order-service/api/orders")

# Environment variable ဖြင့်လည်း ရနိုင်ခြင်း (auto-injected by K8s)
import os
order_host = os.getenv("ORDER_SERVICE_SERVICE_HOST")  # ClusterIP
order_port = os.getenv("ORDER_SERVICE_SERVICE_PORT")   # Port
```

---

## ၂၀.၅ Global Server Load Balancing (GeoDNS)

Multi-region deployment တွင် user ကို အနီးဆုံး region ရှိ server သို့ route လုပ်ရန် **GeoDNS** (Geographic DNS) ကို အသုံးပြုပါသည်။

```
                        +-------------+
                        |   GeoDNS    |
                        | (Route 53)  |
                        +------+------+
                               |
              +----------------+----------------+
              |                |                |
      +-------v------+ +------v-------+ +------v-------+
      | US-East      | | EU-West      | | Asia-Pacific |
      | Region       | | Region       | | Region       |
      | LB + Services| | LB + Services| | LB + Services|
      +--------------+ +--------------+ +--------------+
```

```python
# AWS Route 53 GeoDNS configuration ဥပမာ (boto3)
import boto3

route53 = boto3.client('route53')

# Geolocation routing policy ဖြင့် DNS record ဖန်တီးခြင်း
response = route53.change_resource_record_sets(
    HostedZoneId='Z1234567890',
    ChangeBatch={
        'Changes': [{
            'Action': 'CREATE',
            'ResourceRecordSet': {
                'Name': 'api.myapp.com',
                'Type': 'A',
                'SetIdentifier': 'asia-pacific',       # Region identifier
                'GeoLocation': {
                    'ContinentCode': 'AS'               # Asia continent
                },
                'AliasTarget': {
                    'HostedZoneId': 'Z9876543210',
                    'DNSName': 'ap-alb.myapp.com',     # Asia-Pacific ALB
                    'EvaluateTargetHealth': True         # Health check ပါ ထည့်သွင်းခြင်း
                }
            }
        }]
    }
)
```

### GSLB Strategies

- **Geographic** - User ၏ တည်နေရာ (IP-based) အလိုက် route လုပ်ခြင်း
- **Latency-based** - Network latency အနည်းဆုံး region သို့ route လုပ်ခြင်း
- **Weighted** - Region တစ်ခုချင်းစီသို့ weight ratio အလိုက် traffic ဖြန့်ခြင်း
- **Failover** - Primary region fail ဖြစ်လျှင် secondary region သို့ အလိုအလျောက် ပြောင်းခြင်း

---

## အဓိက အချက်များ (Key Takeaways)

- Client-side discovery သည် latency နိမ့်သော်လည်း client complexity မြင့်သည်။ Server-side discovery သည် ပိုရိုးရှင်းသည်
- Service Registry (Consul, Eureka, etcd) သည် service instances များ၏ health status နှင့် location ကို track လုပ်သည်
- Load balancing strategy ကို use case အလိုက် ရွေးချယ်ပါ - Round Robin (general), Least Connections (long-lived), Consistent Hash (cache affinity)
- Kubernetes ၏ built-in DNS discovery သည် cluster အတွင်း service-to-service communication အတွက် အလွယ်ကူဆုံး နည်းလမ်း ဖြစ်သည်
- GeoDNS ဖြင့် global users များကို nearest region သို့ route လုပ်ပြီး latency လျှော့ချနိုင်သည်
