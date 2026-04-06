# အခန်း ၂၂: Observability (စောင့်ကြည့်နိုင်မှု)

## နိဒါန်း

"You can't fix what you can't see." - Distributed systems တွင် request တစ်ခုသည် services ဆယ်ခုကျော် ဖြတ်သန်းနိုင်ပြီး failure root cause ကို ရှာဖွေခြင်းသည် ထောင်ချောက်ထဲတွင် အပ်ရှာသကဲ့သို့ ဖြစ်နိုင်ပါသည်။ **Observability** ဆိုသည်မှာ system ၏ internal state ကို external outputs (logs, metrics, traces) များဖြင့် နားလည်နိုင်ခြင်း ဖြစ်သည်။ Monitoring သည် "known unknowns" (ဘာတွေ fail ဖြစ်နိုင်လဲ ကြိုသိထားပြီး စောင့်ကြည့်ခြင်း) ကိုသာ ကိုင်တွယ်နိုင်သော်လည်း Observability သည် "unknown unknowns" (ကြိုမသိနိုင်သော ပြဿနာများ) ကိုပါ ရှာဖွေနိုင်စွမ်း ပေးသည်။ ဤအခန်းတွင် observability ၏ three pillars နှင့် SLO/SLA/error budget concepts များကို အသေးစိတ် လေ့လာပါမည်။

---

## ၂၂.၁ Three Pillars: Logs, Metrics, Traces

Observability ၏ အဓိက မဏ္ဍိုင် ၃ ခု ရှိပါသည်။

```
+----------------------------------------------------------------+
|                    Observability                                |
+----------------------------------------------------------------+
|                                                                |
|   +----------+      +-----------+      +------------------+    |
|   |  LOGS    |      | METRICS   |      |    TRACES        |    |
|   |          |      |           |      |                  |    |
|   | What     |      | How much/ |      | Request flow     |    |
|   | happened |      | How many  |      | across services  |    |
|   |          |      |           |      |                  |    |
|   | Event-   |      | Numeric   |      | Distributed      |    |
|   | based    |      | time-     |      | call chain       |    |
|   | detail   |      | series    |      | visualization    |    |
|   +----------+      +-----------+      +------------------+    |
|                                                                |
+----------------------------------------------------------------+
```

- **Logs** - Event-based records။ ဘာဖြစ်ခဲ့သည်ကို အသေးစိတ် မှတ်တမ်းတင်ခြင်း
- **Metrics** - Numeric time-series data။ System ၏ health ကို numbers ဖြင့် တိုင်းတာခြင်း (CPU%, request count, error rate)
- **Traces** - Request တစ်ခု services များကို ဖြတ်သန်းသွားသည့် path ကို track လုပ်ခြင်း

ဤ ၃ ခု ပေါင်းစပ်ခြင်းဖြင့်သာ system ၏ behavior ကို ပြည့်စုံစွာ နားလည်နိုင်ပါသည်။

---

## ၂၂.၂ Distributed Tracing (OpenTelemetry, Jaeger, Zipkin)

Distributed tracing သည် request တစ်ခု services များကို ဖြတ်သန်းသွားသည့် journey ကို visualize လုပ်ပေးသည်။

```
Trace ID: abc-123-def-456

|----- API Gateway (50ms) ------|
    |--- Auth Service (10ms) --|
    |--- Order Service (35ms) ----------------------|
         |--- Inventory Service (15ms) --|
         |--- Payment Service (20ms) ----------|
              |--- Bank API (18ms) ---|
```

### OpenTelemetry (OTel)

OpenTelemetry သည် observability data (traces, metrics, logs) collection အတွက် vendor-neutral open standard ဖြစ်ပါသည်။

```python
# OpenTelemetry setup - tracing configuration
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.flask import FlaskInstrumentor

# Tracer provider setup
provider = TracerProvider()

# Jaeger exporter - trace data ကို Jaeger သို့ ပို့ခြင်း
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger-agent",   # Jaeger agent address
    agent_port=6831                    # Jaeger agent port
)

# Batch processor - performance အတွက် batch ဖြင့် export လုပ်ခြင်း
provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
trace.set_tracer_provider(provider)

# Auto-instrumentation - requests library ကို auto-trace လုပ်ခြင်း
RequestsInstrumentor().instrument()

# Flask app instrumentation
from flask import Flask
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)

# Custom span ဖန်တီးခြင်း
tracer = trace.get_tracer("order-service")

@app.route("/api/orders", methods=["POST"])
def create_order():
    # Parent span - order creation process
    with tracer.start_as_current_span("create_order") as span:
        span.set_attribute("order.customer_id", customer_id)
        
        # Child span - inventory check
        with tracer.start_as_current_span("check_inventory") as child:
            child.set_attribute("product.id", product_id)
            inventory = requests.get(f"http://inventory-service/check/{product_id}")
        
        # Child span - payment processing
        with tracer.start_as_current_span("process_payment") as child:
            child.set_attribute("payment.amount", amount)
            payment = requests.post("http://payment-service/pay", json=payload)
        
        return {"order_id": order_id, "status": "created"}
```

### Trace Context Propagation

Services များကြား trace context ကို HTTP headers ဖြင့် propagate လုပ်ပါသည်။ W3C Trace Context standard ကို အသုံးပြုသည်။

```
HTTP Headers:
traceparent: 00-abc123def456-span789-01
tracestate: vendor1=value1

Format: version-trace_id-parent_span_id-trace_flags
```

---

## ၂၂.၃ Structured Logging & Correlation IDs

### Structured Logging

Traditional text logs ("Order created for user 123") အစား JSON format ဖြင့် structured logs ရေးခြင်းဖြင့် search နှင့် analysis ကို ပိုမိုလွယ်ကူစေသည်။

```python
import structlog
import uuid

# Structlog configuration
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),    # ISO timestamp
        structlog.processors.add_log_level,              # Log level ထည့်ခြင်း
        structlog.processors.JSONRenderer()              # JSON output
    ]
)

logger = structlog.get_logger()

# Correlation ID middleware - request တိုင်းတွင် unique ID ထည့်ခြင်း
def correlation_id_middleware(request):
    # Upstream service မှ propagate လာသော ID ကို ယူခြင်း (မရှိလျှင် အသစ်ဖန်တီးခြင်း)
    correlation_id = request.headers.get("X-Correlation-ID", str(uuid.uuid4()))
    
    # Logger ထဲ correlation ID bind လုပ်ခြင်း
    request_logger = logger.bind(
        correlation_id=correlation_id,
        service="order-service",
        request_path=request.path,
        request_method=request.method
    )
    return correlation_id, request_logger

# Structured log output ဥပမာ
# {
#   "timestamp": "2026-04-06T10:30:00Z",
#   "level": "info",
#   "correlation_id": "abc-123-def",
#   "service": "order-service",
#   "event": "order_created",
#   "order_id": "ord-456",
#   "customer_id": "cust-789",
#   "amount": 99.99
# }
```

### Correlation ID Flow

```
Client Request
  |  X-Correlation-ID: req-abc-123
  v
API Gateway (log: correlation_id=req-abc-123, event=request_received)
  |  X-Correlation-ID: req-abc-123  (propagate)
  v
Order Service (log: correlation_id=req-abc-123, event=order_created)
  |  X-Correlation-ID: req-abc-123  (propagate)
  v
Payment Service (log: correlation_id=req-abc-123, event=payment_processed)
```

ဤနည်းဖြင့် correlation ID တစ်ခုဖြင့် request journey တစ်ခုလုံး၏ logs အားလုံးကို ရှာဖွေနိုင်ပါသည်။

---

## ၂၂.၄ Metrics & Alerting (Prometheus, Grafana)

### Prometheus

Prometheus သည် pull-based metrics collection system ဖြစ်ပြီး time-series database တွင် metrics များကို သိမ်းဆည်းသည်။

```
+----------+     pull      +------------+     query     +---------+
| Service  | <------------ | Prometheus | <------------ | Grafana |
| /metrics |   (scrape)    | (TSDB)     |   (PromQL)   | (UI)    |
+----------+               +-----+------+               +---------+
                                  |
                                  v  alert rules
                           +------+------+
                           | AlertManager|  --> Slack, PagerDuty, Email
                           +-------------+
```

```python
# Prometheus metrics expose လုပ်ခြင်း (Python)
from prometheus_client import Counter, Histogram, Gauge, start_http_server

# Counter - တိုးလာသာ metric (request count, error count)
REQUEST_COUNT = Counter(
    'http_requests_total',                    # Metric name
    'Total HTTP requests',                     # Description
    ['method', 'endpoint', 'status_code']      # Labels
)

# Histogram - distribution metric (latency, response size)
REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency in seconds',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]  # Bucket boundaries
)

# Gauge - အတက်အကျ ရှိသော metric (active connections, queue size)
ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Number of active connections'
)

# Metrics middleware
import time

def metrics_middleware(request, handler):
    """Request တိုင်းအတွက် metrics record လုပ်ခြင်း"""
    start_time = time.time()
    ACTIVE_CONNECTIONS.inc()  # Connection count တိုးခြင်း
    
    try:
        response = handler(request)
        status = response.status_code
    except Exception:
        status = 500
        raise
    finally:
        duration = time.time() - start_time
        REQUEST_COUNT.labels(
            method=request.method,
            endpoint=request.path,
            status_code=str(status)
        ).inc()                                    # Request count တိုးခြင်း
        REQUEST_LATENCY.labels(
            method=request.method,
            endpoint=request.path
        ).observe(duration)                        # Latency record လုပ်ခြင်း
        ACTIVE_CONNECTIONS.dec()                    # Connection count လျှော့ခြင်း
    
    return response

# Metrics server start လုပ်ခြင်း (port 8000)
start_http_server(8000)  # /metrics endpoint ကို expose လုပ်ခြင်း
```

### PromQL Alert Rules ဥပမာ

```yaml
# Prometheus alert rules
groups:
  - name: service_alerts
    rules:
      # Error rate 5% ကျော်လျှင် alert
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          > 0.05
        for: 5m                    # ၅ မိနစ် ဆက်တိုက် ဖြစ်နေလျှင်
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 5% for 5 minutes"

      # P99 latency 2 စက္ကန့် ကျော်လျှင် alert
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            rate(http_request_duration_seconds_bucket[5m])
          ) > 2.0
        for: 10m
        labels:
          severity: warning
```

---

## ၂၂.၅ SLI, SLO, SLA & Error Budgets

### အဓိပ္ပာယ်ဖွင့်ဆိုချက်များ

```
+-------+------------------------------------------+----------------------+
| Term  | အဓိပ္ပာယ်                                 | ဥပမာ                 |
+-------+------------------------------------------+----------------------+
| SLI   | Service Level Indicator                  | Request success rate |
|       | တိုင်းတာနိုင်သော metric                    | = 99.95%             |
+-------+------------------------------------------+----------------------+
| SLO   | Service Level Objective                  | Availability >= 99.9%|
|       | Internal target (goal)                   | P99 latency < 200ms  |
+-------+------------------------------------------+----------------------+
| SLA   | Service Level Agreement                  | 99.9% uptime or      |
|       | Customer နှင့် ချုပ်ဆိုသော စာချုပ်          | credit refund        |
+-------+------------------------------------------+----------------------+
```

### Error Budget

Error budget = 100% - SLO target။ ဥပမာ SLO = 99.9% ဖြစ်လျှင် error budget = 0.1% ဖြစ်သည်။

```
30-day Error Budget (SLO = 99.9%):
Total minutes = 30 * 24 * 60 = 43,200 minutes
Error budget  = 43,200 * 0.001 = 43.2 minutes of downtime allowed

+--------------------------------------------------+
|  Error Budget Consumption (30 days)               |
|  [=============================          ] 68%    |
|                                                   |
|  Budget: 43.2 min | Used: 29.4 min | Left: 13.8  |
|                                                   |
|  Status: WARNING - slow down deployments          |
+--------------------------------------------------+
```

```python
# Error budget calculator
class ErrorBudget:
    def __init__(self, slo_target, window_days=30):
        """
        slo_target: SLO target (ဥပမာ 0.999 = 99.9%)
        window_days: Error budget window (ရက်)
        """
        self.slo_target = slo_target
        self.window_minutes = window_days * 24 * 60
        self.total_budget = self.window_minutes * (1 - slo_target)

    def remaining_budget(self, downtime_minutes):
        """ကျန်ရှိသော error budget (မိနစ်)"""
        remaining = self.total_budget - downtime_minutes
        percentage = (remaining / self.total_budget) * 100
        return {
            "total_budget_minutes": round(self.total_budget, 2),
            "used_minutes": downtime_minutes,
            "remaining_minutes": round(remaining, 2),
            "remaining_percentage": round(percentage, 2),
            "can_deploy": percentage > 20  # Budget 20% ကျော်ကျန်လျှင် deploy ခွင့်ပြုခြင်း
        }

# ဥပမာ - 99.9% SLO
budget = ErrorBudget(slo_target=0.999, window_days=30)
status = budget.remaining_budget(downtime_minutes=29.4)
print(status)
# {'total_budget_minutes': 43.2, 'used_minutes': 29.4, 
#  'remaining_minutes': 13.8, 'remaining_percentage': 31.94, 'can_deploy': True}
```

---

## ၂၂.၆ Continuous Profiling

Continuous profiling သည် production environment တွင် application ၏ CPU, memory, I/O usage ကို အမြဲတမ်း profile လုပ်ခြင်း ဖြစ်ပါသည်။ Metrics ဖြင့် "slow ဖြစ်နေသည်" ကို သိနိုင်သော်လည်း profiling ဖြင့် "ဘာကြောင့် slow ဖြစ်နေသည်" (ဘယ် function, ဘယ် code line) ကို သိနိုင်ပါသည်။

```
Flame Graph (CPU Profile):
+----------------------------------------------------------+
| main()                                                    |
|  +---------------------+  +----------------------------+ |
|  | handleRequest()     |  | processBackground()        | |
|  |  +-------+ +------+ |  |  +----------+ +---------+  | |
|  |  |parseJSON| |query|| |  |  |serialize | |compress |  | |
|  |  | 15%    | |DB   || |  |  | 8%       | | 12%     |  | |
|  |  +-------+ | 45%  || |  |  +----------+ +---------+  | |
|  |            +------+ | |  |                            | |
|  +---------------------+ |  +----------------------------+ |
+----------------------------------------------------------+
```

```python
# Pyroscope continuous profiling setup
# pip install pyroscope-io

import pyroscope

# Pyroscope agent start လုပ်ခြင်း
pyroscope.configure(
    application_name="order-service",      # Application name
    server_address="http://pyroscope:4040", # Pyroscope server
    sample_rate=100,                        # Sample rate (Hz)
    tags={
        "region": "ap-southeast-1",
        "env": "production"
    }
)

# Manual profiling scope
with pyroscope.tag_wrapper({"endpoint": "/api/orders"}):
    # ဤ block အတွင်း code ကို profile လုပ်ခြင်း
    process_order(order_data)
```

### Profiling Tools နှိုင်းယှဉ်ခြင်း

| Tool | Language Support | Overhead | Storage |
|------|-----------------|----------|---------|
| Pyroscope | Go, Python, Java, Ruby | ~2% CPU | Local/Cloud |
| Parca | Go, C++, Rust (eBPF) | < 1% CPU | Object Store |
| Datadog Profiling | Java, Python, Go, .NET | ~2% CPU | Datadog Cloud |

---

## အဓိက အချက်များ (Key Takeaways)

- Observability ၏ three pillars - **Logs** (ဘာဖြစ်ခဲ့လဲ), **Metrics** (ဘယ်လောက်ဖြစ်လဲ), **Traces** (ဘယ်ကနေ ဘယ်ကို သွားလဲ) - ၃ ခုလုံး လိုအပ်သည်
- **OpenTelemetry** သည် vendor-neutral observability standard ဖြစ်ပြီး traces, metrics, logs အတွက် unified API ပေးသည်
- **Structured logging** (JSON) နှင့် **correlation IDs** ဖြင့် distributed request ကို track လုပ်နိုင်သည်
- **Prometheus + Grafana** သည် metrics collection နှင့် visualization အတွက် de facto standard ဖြစ်သည်
- **SLO/Error Budget** ဖြင့် reliability နှင့် velocity ကြား balance ကို data-driven ဖြင့် ဆုံးဖြတ်နိုင်သည်
- **Continuous Profiling** ဖြင့် production ရှိ performance bottleneck ကို code level အထိ ရှာဖွေနိုင်သည်
