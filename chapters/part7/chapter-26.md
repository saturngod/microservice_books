# အခန်း ၂၆ — Multi-Region နှင့် Disaster Recovery

## နိဒါန်း

Production system တစ်ခု build ပြီး "works" ဆိုသည်နှင့် မပြီးသေးပါ။ Datacenter တစ်ခု down ဖြစ်သွားသောအခါ၊ cloud provider region တစ်ခု outage ဖြစ်သောအခါ၊ database corruption ဖြစ်သောအခါ — system ဆက်လုပ်ဆောင်နိုင်မည်လား ဆိုသည်မှာ Disaster Recovery planning ၏ core question ဖြစ်သည်။ ဤအခန်းတွင် Active-Active vs Active-Passive architectures, cross-region data replication, RTO/RPO concepts, Chaos Engineering practices, နှင့် Runbooks/Game Days တို့ကို အသေးစိတ် လေ့လာသွားမည်ဖြစ်သည်။

---

## ၂၆.၁ Active-Active vs Active-Passive

### Active-Active Architecture

Active-Active architecture တွင် regions နှစ်ခု (သို့မဟုတ် ထို့ထက်ပိုသော) သည် traffic ကို simultaneously serve ပြုလုပ်သည်။ Region တစ်ခု down ဖြစ်ပါက remaining regions က traffic ကို absorb ပြုလုပ်သည်။

```
Active-Active Architecture:
┌──────────────────────────────────────────────────────────────┐
│                    Global DNS / CDN                           │
│              (GeoDNS / Latency-based routing)                │
└────────────┬─────────────────────────┬───────────────────────┘
             │                         │
    ┌────────▼─────────┐     ┌────────▼─────────┐
    │  Region: US-East  │     │  Region: EU-West  │
    │  (Active)         │     │  (Active)         │
    │                   │     │                   │
    │  ┌─────────────┐  │     │  ┌─────────────┐  │
    │  │ K8s Cluster │  │     │  │ K8s Cluster │  │
    │  │ (Services)  │  │     │  │ (Services)  │  │
    │  └─────────────┘  │     │  └─────────────┘  │
    │  ┌─────────────┐  │     │  ┌─────────────┐  │
    │  │  Database   │◄─┼─────┼─►│  Database   │  │
    │  │  (Primary)  │  │ Sync│  │  (Primary)  │  │
    │  └─────────────┘  │     │  └─────────────┘  │
    └───────────────────┘     └───────────────────┘

Both regions serve traffic simultaneously
Cross-region data sync: async replication (eventual consistency)
Failover: DNS failover — remaining region absorbs all traffic
```

### Active-Passive Architecture

Active-Passive architecture တွင် Primary region သာ traffic serve ပြုလုပ်ပြီး Passive region သည် standby mode ဖြင့် ရှိနေသည်။ Primary down ဖြစ်ပါက Passive ကို promote ပြုလုပ်သည်။

```
Active-Passive Architecture:
┌──────────────────────────────────────────────────────────────┐
│                        Global DNS                            │
└────────────┬─────────────────────────┬───────────────────────┘
             │ (all traffic)           │ (failover only)
    ┌────────▼─────────┐     ┌────────▼─────────┐
    │  Region: US-East  │     │  Region: US-West  │
    │  (ACTIVE)         │     │  (PASSIVE/Standby)│
    │                   │     │                   │
    │  ┌─────────────┐  │     │  ┌─────────────┐  │
    │  │ K8s Cluster │  │     │  │ K8s Cluster │  │
    │  │ (Running)   │  │     │  │ (Scaled down│  │
    │  └─────────────┘  │     │  │  or warm)   │  │
    │  ┌─────────────┐  │     │  └─────────────┘  │
    │  │  Database   │──┼─────┼─>│  Database   │  │
    │  │  (Primary)  │  │ Repl│  │  (Replica)  │  │
    │  └─────────────┘  │     │  └─────────────┘  │
    └───────────────────┘     └───────────────────┘

Primary region serves all traffic
Passive region: standby (warm/cold)
Failover: DNS switch + DB promote (minutes to hours)
```

### Active-Active vs Active-Passive Comparison

| Aspect | Active-Active | Active-Passive |
|---|---|---|
| **Failover Time** | Seconds (DNS TTL) | Minutes to Hours |
| **Data Consistency** | Eventual (conflict possible) | Strong (single writer) |
| **Cost** | High (2x resources always running) | Medium (standby cheaper) |
| **Complexity** | High (conflict resolution) | Medium |
| **Latency** | Lower (geo-routing) | Higher (single region) |
| **Use Case** | Global apps, zero-downtime required | Regional apps, cost-sensitive |

### Data Conflict Resolution (Active-Active)

Active-Active တွင် regions နှစ်ခုမှ simultaneous writes ဖြစ်နိုင်သောကြောင့် conflict resolution strategy လိုအပ်သည်။

```python
# Conflict resolution strategies
class ConflictResolver:
    """Active-Active architecture တွင် data conflict ဖြေရှင်းသည်"""

    def last_writer_wins(self, local_record, remote_record):
        """Last Writer Wins (LWW) — timestamp comparison
        ရိုးရှင်းသော approach — data loss ဖြစ်နိုင်သည်"""
        if local_record.updated_at >= remote_record.updated_at:
            return local_record
        return remote_record

    def merge_records(self, local_record, remote_record):
        """Field-level merge — conflict ဖြစ်သော fields ကိုသာ resolve
        ပိုမှန်ကန်သော approach — implementation ပိုရှုပ်ထွေးသည်"""
        merged = {}
        for field in local_record.fields():
            local_val = getattr(local_record, field)
            remote_val = getattr(remote_record, field)

            if local_val == remote_val:
                # No conflict — same value
                merged[field] = local_val
            else:
                # Conflict — field type ပေါ်မူတည်ပြီး resolve
                if field == "balance":
                    # Numeric fields: CRDT counter merge
                    merged[field] = local_val + remote_val - local_record.base_value
                elif field == "tags":
                    # Set fields: union merge
                    merged[field] = set(local_val) | set(remote_val)
                else:
                    # Default: LWW for this field
                    merged[field] = self.last_writer_wins(
                        local_record, remote_record
                    )
        return merged

    def vector_clock_resolve(self, local_record, remote_record):
        """Vector Clock — causal ordering ဖြင့် conflict detect
        Dynamo-style: concurrent writes detect ပြီး application-level resolve"""
        local_vc = local_record.vector_clock
        remote_vc = remote_record.vector_clock

        if local_vc.dominates(remote_vc):
            return local_record      # Local is newer
        elif remote_vc.dominates(local_vc):
            return remote_record     # Remote is newer
        else:
            # Concurrent writes — true conflict
            # Application-specific merge logic required
            return self.merge_records(local_record, remote_record)
```

---

## ၂၆.၂ Cross-Region Data Replication

### Replication Strategies

```
Replication Strategies:
┌──────────────────────────────────────────────────────────────┐
│  1. Synchronous Replication                                  │
│     Write ──> Primary ──> Replica (wait for ACK) ──> Commit  │
│     + Strong consistency                                     │
│     - High latency (cross-region RTT: 50-200ms)              │
│     - Reduced throughput                                     │
│                                                              │
│  2. Asynchronous Replication                                 │
│     Write ──> Primary ──> Commit ──> Replica (background)    │
│     + Low latency                                            │
│     + High throughput                                        │
│     - Replication lag (data loss risk on failure)             │
│                                                              │
│  3. Semi-Synchronous Replication                             │
│     Write ──> Primary ──> 1 Replica ACK ──> Commit           │
│     + Balance of consistency and performance                 │
│     - Slightly higher latency than async                     │
└──────────────────────────────────────────────────────────────┘
```

### Database Replication Configurations

```yaml
# PostgreSQL cross-region streaming replication
# Primary (US-East) postgresql.conf
wal_level = replica
max_wal_senders = 10
synchronous_standby_names = ''           # Async replication
archive_mode = on
archive_command = 'aws s3 cp %p s3://wal-archive/%f'

# Standby (EU-West) recovery.conf
primary_conninfo = 'host=primary-us-east.db port=5432 user=replicator'
restore_command = 'aws s3 cp s3://wal-archive/%f %p'
recovery_target_timeline = 'latest'
```

```python
# Application-level cross-region event replication
# Event Sourcing pattern ဖြင့် region ကြား data sync
import json
from kafka import KafkaProducer, KafkaConsumer

class CrossRegionReplicator:
    """Kafka ဖြင့် cross-region event replication ပြုလုပ်သည်"""

    def __init__(self, local_region, remote_regions):
        self.local_region = local_region
        self.remote_regions = remote_regions

        # Local Kafka producer — events publish
        self.producer = KafkaProducer(
            bootstrap_servers=f"kafka.{local_region}.internal:9092",
            value_serializer=lambda v: json.dumps(v).encode("utf-8"),
            # Idempotent producer — duplicate prevention
            enable_idempotence=True,
            acks="all",
        )

    def publish_event(self, event_type, data):
        """Local event publish — remote regions ကို replicate ပြုလုပ်မည်"""
        event = {
            "event_id": str(uuid.uuid4()),
            "event_type": event_type,
            "source_region": self.local_region,
            "timestamp": datetime.utcnow().isoformat(),
            "data": data,
        }

        # Cross-region replication topic သို့ publish
        self.producer.send(
            topic="cross-region-events",
            key=data.get("entity_id", "").encode(),
            value=event,
        )
        self.producer.flush()
        return event["event_id"]

    def consume_remote_events(self):
        """Remote regions မှ events consume ပြုလုပ်ပြီး local DB update"""
        consumer = KafkaConsumer(
            "cross-region-events",
            bootstrap_servers=f"kafka.{self.local_region}.internal:9092",
            group_id=f"replicator-{self.local_region}",
            value_deserializer=lambda v: json.loads(v.decode("utf-8")),
        )

        for message in consumer:
            event = message.value
            # Own region events ကို skip (duplicate prevention)
            if event["source_region"] == self.local_region:
                continue

            # Idempotency check — event_id ဖြင့် duplicate detect
            if self._is_processed(event["event_id"]):
                continue

            # Local database update ပြုလုပ်သည်
            self._apply_event(event)
            self._mark_processed(event["event_id"])
```

---

## ၂၆.၃ RTO vs RPO

### Definitions

```
RTO vs RPO Timeline:
                    ← RPO →          ← RTO →
                    │                 │
  ────────────────╳─┼─────────────────┼──────────────►
  Last backup     Disaster           Recovery      Time
  (or last        occurs             complete
   sync point)

RPO (Recovery Point Objective):
  "ဘယ်လောက် data ဆုံးရှုံးခံနိုင်လဲ"
  RPO = 0: zero data loss (synchronous replication)
  RPO = 1 hour: 1 hour worth of data loss acceptable

RTO (Recovery Time Objective):
  "ဘယ်လောက်ကြာပြီး ပြန်ကောင်းရမလဲ"
  RTO = 0: zero downtime (active-active)
  RTO = 4 hours: 4 hours downtime acceptable
```

### RTO/RPO Tiers

| Tier | RPO | RTO | Strategy | Cost |
|---|---|---|---|---|
| **Tier 1: Mission Critical** | ~0 | < 1 min | Active-Active, sync replication | Very High |
| **Tier 2: Business Critical** | < 1 hour | < 15 min | Active-Passive (warm), async repl | High |
| **Tier 3: Important** | < 4 hours | < 1 hour | Pilot Light (minimal standby) | Medium |
| **Tier 4: Non-Critical** | < 24 hours | < 24 hours | Backup & Restore | Low |

### Infrastructure by RTO/RPO

```
DR Strategy Cost vs Recovery Speed:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Cost ↑                                                     │
│       │  ★ Active-Active                                    │
│       │     (RTO: ~0, RPO: ~0)                              │
│       │                                                     │
│       │     ★ Warm Standby                                  │
│       │       (RTO: 15min, RPO: minutes)                    │
│       │                                                     │
│       │        ★ Pilot Light                                │
│       │          (RTO: 1hr, RPO: hours)                     │
│       │                                                     │
│       │           ★ Backup/Restore                          │
│       │             (RTO: 24hr, RPO: 24hr)                  │
│       └──────────────────────────────────────────────────► │
│                            Recovery Speed                   │
└─────────────────────────────────────────────────────────────┘
```

```yaml
# Multi-region Kubernetes deployment — Warm Standby
# Primary region: full deployment
# DR region: scaled-down deployment (warm standby)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    region: us-east-1
spec:
  replicas: 10              # Primary: full capacity
  # ... (full deployment spec)
---
# DR Region deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    region: eu-west-1
spec:
  replicas: 2               # DR: minimal (warm standby)
  # Failover script scales up to 10
  # ... (same deployment spec, different replicas)
```

```python
# Failover automation script
import boto3
import subprocess

class FailoverOrchestrator:
    """Region failover orchestrate ပြုလုပ်သည်"""

    def __init__(self, primary_region, dr_region):
        self.primary = primary_region
        self.dr = dr_region
        self.route53 = boto3.client("route53")

    def execute_failover(self):
        """DR region သို့ failover ပြုလုပ်သည်"""
        print(f"Initiating failover: {self.primary} → {self.dr}")

        # Step 1: DR region services scale up
        self._scale_up_dr_region()

        # Step 2: Database promote (replica → primary)
        self._promote_database()

        # Step 3: DNS update — traffic ကို DR region သို့ redirect
        self._update_dns()

        # Step 4: Health check verify
        self._verify_health()

        print("Failover complete")

    def _scale_up_dr_region(self):
        """DR region Kubernetes deployments scale up"""
        subprocess.run([
            "kubectl", "--context", f"k8s-{self.dr}",
            "scale", "deployment", "--all",
            "--replicas=10", "-n", "production"
        ], check=True)

    def _promote_database(self):
        """DR region database ကို primary အဖြစ် promote"""
        rds = boto3.client("rds", region_name=self.dr)
        rds.promote_read_replica(
            DBInstanceIdentifier=f"orders-db-{self.dr}-replica"
        )
        # Wait for promotion complete
        waiter = rds.get_waiter("db_instance_available")
        waiter.wait(DBInstanceIdentifier=f"orders-db-{self.dr}-replica")

    def _update_dns(self):
        """Route53 DNS record update — DR region IP သို့ switch"""
        self.route53.change_resource_record_sets(
            HostedZoneId="Z1234567890",
            ChangeBatch={
                "Changes": [{
                    "Action": "UPSERT",
                    "ResourceRecordSet": {
                        "Name": "api.myapp.com",
                        "Type": "A",
                        "AliasTarget": {
                            "HostedZoneId": "Z0987654321",
                            "DNSName": f"alb-{self.dr}.amazonaws.com",
                            "EvaluateTargetHealth": True,
                        }
                    }
                }]
            }
        )
```

---

## ၂၆.၄ Chaos Engineering

### Chaos Engineering Principles

Chaos Engineering သည် production system ၏ resilience ကို verify ပြုလုပ်ရန် controlled experiments ပြုလုပ်ခြင်းဖြစ်သည်။ "System ကို ရိုက်ခွဲပြီး ဘယ်လို react လုပ်သလဲ ကြည့်ခြင်း" ဟု ရိုးရှင်းစွာ ဖော်ပြနိုင်သည်။

```
Chaos Engineering Process:
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ 1. Steady     │───>│ 2. Hypothesis │───>│ 3. Experiment │
│    State      │    │               │    │    Design     │
│ "Normal KPIs" │    │ "System should│    │ "Kill Pod X"  │
│               │    │  handle X"    │    │               │
└───────────────┘    └───────────────┘    └───────┬───────┘
                                                   │
┌───────────────┐    ┌───────────────┐    ┌───────▼───────┐
│ 6. Improve    │◄───│ 5. Analyze    │◄───│ 4. Run        │
│    System     │    │    Results    │    │    Experiment  │
│ "Fix found    │    │ "Matched     │    │ "Monitor KPIs"│
│  weaknesses"  │    │  hypothesis?"│    │               │
└───────────────┘    └───────────────┘    └───────────────┘
```

### Chaos Tools & Experiments

```yaml
# Chaos Mesh — Kubernetes-native chaos engineering
# Pod kill experiment — service resilience test
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: order-service-pod-kill
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one                       # Pod 1 ခုသာ kill
  selector:
    namespaces:
      - production
    labelSelectors:
      app: order-service
  # Scheduler — every day at 2pm (business hours — intentional)
  scheduler:
    cron: "0 14 * * 1-5"         # Weekdays 2 PM
  duration: "30s"
---
# Network delay experiment — cross-service latency impact test
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: payment-network-delay
  namespace: chaos-testing
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: payment-service
  delay:
    latency: "500ms"              # 500ms latency inject
    jitter: "100ms"               # 100ms variation
    correlation: "50"
  duration: "10m"                 # 10 min duration
---
# CPU stress experiment — resource saturation test
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: order-service-cpu-stress
spec:
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: order-service
  stressors:
    cpu:
      workers: 4                  # 4 CPU-intensive workers
      load: 80                    # 80% CPU load
  duration: "15m"
```

```python
# Chaos experiment automation — Litmus/custom
class ChaosExperiment:
    """Chaos experiment define + run + analyze"""

    def __init__(self, name, target_service, blast_radius="one_pod"):
        self.name = name
        self.target_service = target_service
        self.blast_radius = blast_radius
        self.steady_state_metrics = {}

    def capture_steady_state(self):
        """Experiment မတိုင်ခင် baseline metrics capture"""
        self.steady_state_metrics = {
            "error_rate": self._query_prometheus(
                f'sum(rate(http_requests_total{{service="{self.target_service}",status=~"5.."}}[5m]))'
                f'/ sum(rate(http_requests_total{{service="{self.target_service}"}}[5m])) * 100'
            ),
            "p99_latency": self._query_prometheus(
                f'histogram_quantile(0.99, rate(http_request_duration_seconds_bucket'
                f'{{service="{self.target_service}"}}[5m]))'
            ),
            "success_rate": self._query_prometheus(
                f'sum(rate(http_requests_total{{service="{self.target_service}",status=~"2.."}}[5m]))'
                f'/ sum(rate(http_requests_total{{service="{self.target_service}"}}[5m])) * 100'
            ),
        }
        print(f"Steady state: {self.steady_state_metrics}")

    def verify_hypothesis(self):
        """Experiment ပြီးနောက် hypothesis verify"""
        current = {
            "error_rate": self._query_prometheus("..."),
            "p99_latency": self._query_prometheus("..."),
            "success_rate": self._query_prometheus("..."),
        }

        # Hypothesis: error rate 1% ထက် မကျော်သင့်
        assert current["error_rate"] < 1.0, \
            f"Error rate too high: {current['error_rate']}%"

        # Hypothesis: latency 2x ထက် မတိုးသင့်
        assert current["p99_latency"] < self.steady_state_metrics["p99_latency"] * 2, \
            f"Latency spike: {current['p99_latency']}s"

        # Hypothesis: success rate 99% အထက် ရှိသင့်
        assert current["success_rate"] > 99.0, \
            f"Success rate dropped: {current['success_rate']}%"

        print("Hypothesis verified - system is resilient!")
```

### Common Chaos Experiments

| Experiment | Target | Expected Behavior |
|---|---|---|
| **Pod Kill** | Random service pod | Auto-restart, no user impact |
| **Network Delay** | Service-to-service | Circuit breaker activates, timeout |
| **Network Partition** | Database connection | Retry + fallback to cache |
| **CPU Stress** | Service container | HPA scales up, latency bounded |
| **Disk Full** | Persistent volume | Alert fires, log rotation |
| **Zone Outage** | Entire AZ | Traffic routes to other AZs |
| **DNS Failure** | Service discovery | Cached DNS, retry with backoff |

---

## ၂၆.၅ Runbooks & Game Days

### Runbooks

Runbook သည် incident response အတွက် step-by-step guide ဖြစ်သည်။ 3AM ရောက်ပြီး on-call engineer alarm ခေါင်းလောင်းမှ ကိုယ်ဘာလုပ်ရမှန်း ချက်ခြင်း သိရန် Runbook ရေးထားသင့်သည်။

```markdown
# Runbook: Order Service High Error Rate

## Alert Condition
- Order Service 5xx error rate > 5% for 5 minutes
- PagerDuty alert: ORDER_SVC_HIGH_ERROR_RATE

## Severity Assessment
- < 10% error rate: SEV-2 (degraded)
- > 10% error rate: SEV-1 (outage)
- > 50% error rate: SEV-0 (critical — executive notification)

## Diagnostic Steps

### Step 1: Verify the alert (2 minutes)
  - Grafana dashboard ကြည့်: https://grafana.internal/d/order-svc
  - Actual error rate confirm ပြုလုပ်: real vs false positive
  - Affected endpoints identify: /api/orders/create vs /api/orders/{id}

### Step 2: Check recent deployments (1 minute)
  - ArgoCD recent syncs: https://argocd.internal/applications/order-service
  - Last deploy: `argocd app history order-service-prod`
  - Recent deploy ဖြစ်ပါက → rollback consider

### Step 3: Check dependencies (3 minutes)
  - Database connection pool: `kubectl exec -it order-svc-xxx -- curl localhost:8080/health/db`
  - Redis connection: `kubectl exec -it order-svc-xxx -- curl localhost:8080/health/cache`
  - Kafka consumer lag: https://grafana.internal/d/kafka-lag
  - Downstream services health: payment-svc, inventory-svc

### Step 4: Check logs (2 minutes)
  - `kubectl logs -l app=order-service -n production --tail=100 | grep ERROR`
  - Kibana: https://kibana.internal/app/discover (filter: service=order-service, level=ERROR)

## Mitigation Actions

### If recent deployment caused it:
  `argocd app rollback order-service-prod`

### If database connection issues:
  1. Check DB connections: `SELECT count(*) FROM pg_stat_activity;`
  2. Kill idle connections: `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state='idle' AND query_start < now() - interval '10 minutes';`
  3. Restart connection pool: `kubectl rollout restart deployment/order-service -n production`

### If downstream service failure:
  1. Check circuit breaker status in Grafana
  2. If circuit open → expected behavior, wait for downstream recovery
  3. If circuit not opening → check circuit breaker configuration

## Escalation
- SEV-1: Notify #incident-response Slack channel
- SEV-0: Page engineering manager + VP Engineering
```

### Game Days

Game Day သည် controlled environment တွင် disaster scenario ကို simulate ပြီး team ၏ response ကို practice ပြုလုပ်ခြင်းဖြစ်သည်။

```
Game Day Planning:
┌─────────────────────────────────────────────────────────────┐
│  Pre-Game Day (1 week before)                               │
│  □ Scenario ရွေးချယ်: "US-East region outage"                │
│  □ Blast radius define: staging environment only (first time)│
│  □ Participants identify: on-call rotation, SRE team         │
│  □ Success criteria define: RTO < 15 min, RPO < 5 min       │
│  □ Rollback plan ready                                       │
│  □ Customer communication template ready                     │
├─────────────────────────────────────────────────────────────┤
│  During Game Day (2-4 hours)                                │
│  □ Briefing: scenario explain, rules                         │
│  □ Inject failure: simulate region outage                    │
│  □ Observe: team response, runbook follow, communication     │
│  □ Track: timeline, decisions, blockers                      │
│  □ Conclude: restore normal state                            │
├─────────────────────────────────────────────────────────────┤
│  Post-Game Day (1 week after)                               │
│  □ Retrospective: what worked, what didn't                   │
│  □ Runbook updates based on findings                         │
│  □ Action items with owners + deadlines                      │
│  □ Schedule next game day                                    │
└─────────────────────────────────────────────────────────────┘
```

### Game Day Scenarios

| Scenario | Injection | Expected Response |
|---|---|---|
| **Region Outage** | DNS failover to DR | Services recover in DR, < 15 min |
| **Database Failure** | Kill primary DB | Replica promote, < 5 min |
| **Network Partition** | Block service-to-service | Circuit breakers activate |
| **Dependency Failure** | Kill payment provider | Graceful degradation, retry later |
| **Data Corruption** | Inject bad data | Validation catches, alert fires |
| **Certificate Expiry** | Expire TLS cert | Auto-renewal or manual rotation |

### Game Day Report Template

```
Game Day Report: Region Failover Test
Date: 2026-04-01
Duration: 2 hours
Participants: SRE Team, Order Team, Platform Team

Scenario: US-East region complete outage
Injection: Killed all K8s nodes in us-east-1

Timeline:
  T+0:00  - Injection started
  T+0:02  - First alert fired (PagerDuty)
  T+0:03  - On-call acknowledged
  T+0:05  - Runbook opened, diagnostic started
  T+0:08  - Decision: initiate failover to eu-west-1
  T+0:12  - DR region scaled up
  T+0:15  - Database promoted in DR region
  T+0:18  - DNS updated, traffic flowing to DR
  T+0:20  - Health checks passing
  T+0:25  - All services healthy in DR

Results:
  Actual RTO: 18 minutes (target: 15 min) ← MISSED
  Actual RPO: 2 minutes (target: 5 min) ← MET
  Data loss: 47 orders (async replication lag)

Findings:
  1. DB promotion took 7 min (expected 3 min) — script needs optimization
  2. Runbook step 4 was unclear — team hesitated
  3. Slack notification template was outdated

Action Items:
  1. [SRE] Optimize DB failover script — target: 3 min (due: Apr 15)
  2. [SRE] Update runbook step 4 with clear decision tree (due: Apr 8)
  3. [Platform] Update Slack templates (due: Apr 10)
  4. [All] Schedule follow-up game day in 4 weeks
```

### Incident Post-Mortem Template

```
Post-Mortem Checklist:
□ Blameless culture — blame people ကို ရှောင်ကြဉ်
□ Timeline documented (minute-by-minute)
□ Root cause identified (5 Whys technique)
□ Impact quantified (affected users, revenue loss, duration)
□ What went well (detection speed, team coordination)
□ What went wrong (slow detection, unclear runbook)
□ Action items with owners + deadlines
□ Runbook updated based on learnings
□ Game day scheduled for identified gaps
```

---

## အဓိက အချက်များ (Key Takeaways)

1. **Active-Active** သည် zero-downtime failover ပေးသော်လည်း data conflict resolution ရှုပ်ထွေးပြီး cost မြင့်သည်။ **Active-Passive** သည် simpler ဖြစ်သော်လည်း failover time ပိုကြာသည်

2. **Cross-region replication** — async (fast, eventual consistency), sync (strong consistency, slow), semi-sync (balanced) — RPO target အလိုက် ရွေးချယ်သင့်သည်

3. **RTO** (ဘယ်နှစ်မိနစ်ကြာပြီး ပြန်ကောင်းမလဲ) နှင့် **RPO** (ဘယ်လောက် data ဆုံးရှုံးခံနိုင်လဲ) ကို business requirements ဖြင့် define ပြုလုပ်ပြီး infrastructure design ပြုလုပ်သည်

4. **Chaos Engineering** — production failures ကို proactively discover ပြုလုပ်ရန် controlled experiments (pod kill, network delay, zone outage) ပြုလုပ်သည်

5. **Runbooks** — 3AM incident ကို effectively handle ရန် step-by-step procedures ရေးထားသင့်ပြီး regularly update ပြုလုပ်သင့်သည်

6. **Game Days** — disaster scenarios ကို simulate ပြုလုပ်ပြီး team readiness verify ကာ runbook gaps, skill gaps ကို discover ပြီး improve ပြုလုပ်သည်

---

> **နောက်အခန်း Preview:** Part 8 တွင် Security patterns — OAuth2, mTLS, secret management, OWASP API Security တို့ကို ဆွေးနွေးသွားမည်ဖြစ်သည်။
