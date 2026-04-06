# အခန်း ၂၅ — Microservices အတွက် CI/CD

## နိဒါန်း

Microservices architecture ၏ အဓိက အားသာချက်တစ်ခုမှာ services များကို **independently deploy** ပြုလုပ်နိုင်ခြင်းဖြစ်သည်။ သို့သော် services ဆယ်ကျော်ရာ ရှိလာသောအခါ deployment pipeline management သည် ရှုပ်ထွေးလာသည်။ ဤအခန်းတွင် independent pipelines, blue-green/canary deployments, feature flags, GitOps (ArgoCD/Flux), testing pyramid, နှင့် contract testing (Pact) တို့ကို production-grade level ဖြင့် အသေးစိတ် ရှင်းလင်းသွားမည်ဖြစ်သည်။

---

## ၂၅.၁ Independent Pipelines per Service

### Monorepo vs Polyrepo

```
Pipeline Strategy Comparison:
┌──────────────────────────────────────────────────────────────┐
│  Polyrepo (Repo per Service)                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ order-svc    │  │ user-svc     │  │ product-svc  │      │
│  │ repo         │  │ repo         │  │ repo         │      │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │      │
│  │  │Pipeline│  │  │  │Pipeline│  │  │  │Pipeline│  │      │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  + Simple pipeline    + Clear ownership                      │
│  - Cross-service changes difficult                           │
├──────────────────────────────────────────────────────────────┤
│  Monorepo (All in One Repo)                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  monorepo/                                            │   │
│  │  ├── services/order-svc/     → Pipeline A (changed)   │   │
│  │  ├── services/user-svc/      → Pipeline B (skip)      │   │
│  │  └── libs/shared/            → ALL pipelines trigger   │   │
│  └──────────────────────────────────────────────────────┘   │
│  + Atomic cross-service changes    + Shared libs easy        │
│  - Path-based trigger complexity                             │
└──────────────────────────────────────────────────────────────┘
```

### GitHub Actions — Path-Based Pipeline

```yaml
# .github/workflows/order-service.yml
name: Order Service CI/CD

on:
  push:
    branches: [main]
    paths:
      - "services/order-svc/**"     # order-svc ပြောင်းမှ trigger
      - "libs/shared/**"            # shared lib ပြောင်လည်း trigger
  pull_request:
    branches: [main]
    paths:
      - "services/order-svc/**"
      - "libs/shared/**"

env:
  SERVICE_NAME: order-service
  IMAGE_REGISTRY: ghcr.io/myorg

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"
      - name: Run unit tests
        working-directory: services/order-svc
        run: |
          go test -v -race -coverprofile=coverage.out ./...
          go tool cover -func=coverage.out
      - name: Run linter
        uses: golangci/golangci-lint-action@v4
        with:
          working-directory: services/order-svc

  contract-test:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - name: Run Pact contract tests
        working-directory: services/order-svc
        run: go test -v -tags=contract ./contracts/...

  build:
    runs-on: ubuntu-latest
    needs: [test, contract-test]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_REGISTRY }}/${{ env.SERVICE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: services/order-svc
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: |
          argocd app set ${{ env.SERVICE_NAME }}-staging \
            --helm-set image.tag=${{ github.sha }}
          argocd app sync ${{ env.SERVICE_NAME }}-staging --wait

  integration-test:
    runs-on: ubuntu-latest
    needs: deploy-staging
    steps:
      - uses: actions/checkout@v4
      - name: Run integration tests
        run: |
          cd tests/integration
          API_URL=https://staging-api.myapp.com go test -v ./...

  deploy-production:
    runs-on: ubuntu-latest
    needs: integration-test
    environment: production         # Manual approval gate
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production (canary)
        run: |
          argocd app set ${{ env.SERVICE_NAME }}-prod \
            --helm-set canary.enabled=true \
            --helm-set canary.weight=10 \
            --helm-set image.tag=${{ github.sha }}
          argocd app sync ${{ env.SERVICE_NAME }}-prod --wait
```

```
Pipeline Flow:
┌──────┐   ┌──────┐   ┌──────────┐   ┌───────┐   ┌─────────┐
│ Push │──>│ Test │──>│ Contract │──>│ Build │──>│ Deploy  │
│ Code │   │ Lint │   │  Tests   │   │ Image │   │ Staging │
└──────┘   └──────┘   └──────────┘   └───────┘   └────┬────┘
                                                        │
                                                 ┌──────▼──────┐
                                                 │ Integration │
                                                 │   Tests     │
                                                 └──────┬──────┘
                                                        │
                                                 ┌──────▼──────┐
                                                 │  Approval   │──> Deploy Prod
                                                 └─────────────┘
```

---

## ၂၅.၂ Blue-Green & Canary Deployments

### Blue-Green Deployment

```
Blue-Green Flow:
Step 1: Blue is live
┌────────┐   100%   ┌──────────┐
│  LB    │ ───────> │ Blue v1  │ ← Live
└────────┘          └──────────┘
                    ┌──────────┐
                    │ Green v2 │ ← Deploy here
                    └──────────┘

Step 2: Switch to Green
┌────────┐   100%   ┌──────────┐
│  LB    │ ───────> │ Green v2 │ ← Live
└────────┘          └──────────┘
                    ┌──────────┐
                    │ Blue v1  │ ← Rollback ready
                    └──────────┘
```

```yaml
# Argo Rollouts — Blue-Green
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  replicas: 5
  strategy:
    blueGreen:
      activeService: order-service-active
      previewService: order-service-preview
      autoPromotionEnabled: false       # Manual verify ပြီးမှ promote
      scaleDownDelaySeconds: 600        # 10 min rollback window
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:v2.0.0
          ports:
            - containerPort: 8080
```

### Canary Deployment

```
Canary Progressive Rollout:
  5% ──> 25% ──> 50% ──> 100%
  (monitor metrics at each step)
  
  Automated analysis: error rate < 1%, latency p99 < 500ms
  Failure: auto-rollback to stable version
```

```yaml
# Argo Rollouts — Canary with automated analysis
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: canary-success-rate
            args:
              - name: service-name
                value: order-service
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
      canaryMetadata:
        labels:
          role: canary
      stableMetadata:
        labels:
          role: stable
---
# AnalysisTemplate — Prometheus metrics ဖြင့် canary verify
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-success-rate
spec:
  metrics:
    - name: success-rate
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{
              service="{{args.service-name}}",
              role="canary", status=~"2.."
            }[5m])) /
            sum(rate(http_requests_total{
              service="{{args.service-name}}",
              role="canary"
            }[5m])) * 100
      successCondition: result[0] >= 99.0
      failureLimit: 3
      interval: 60s
```

| Strategy | Rollback | Risk | Cost | Best For |
|---|---|---|---|---|
| **Rolling** | Slow | Medium | Low | Minor updates |
| **Blue-Green** | Instant | Low | High (2x) | Critical services |
| **Canary** | Fast | Very Low | Medium | High-traffic |

---

## ၂၅.၃ Feature Flags

Feature flags ဖြင့် deployment နှင့် release ကို decouple ပြုလုပ်နိုင်သည်။

```
Feature Flag Types:
  Release Toggle    — Gradual rollout (5% → 25% → 100%)
  Experiment Toggle — A/B testing
  Ops Toggle        — Kill switch (instant disable)
  Permission Toggle — Premium features
```

```python
# Feature flag implementation
import hashlib

class FeatureFlagService:
    """Percentage rollout + user targeting support"""

    def __init__(self, flag_store):
        self.flag_store = flag_store

    def is_enabled(self, flag_name, user_id=None, default=False):
        flag = self.flag_store.get(flag_name)
        if not flag:
            return default

        # Kill switch
        if not flag.get("enabled", False):
            return False

        # Allowlist/Blocklist
        if user_id and user_id in flag.get("allowlist", []):
            return True
        if user_id and user_id in flag.get("blocklist", []):
            return False

        # Percentage rollout — deterministic hash
        percentage = flag.get("percentage", 0)
        if user_id and percentage > 0:
            # Same user always gets same result
            hash_val = int(hashlib.md5(
                f"{flag_name}:{user_id}".encode()
            ).hexdigest(), 16) % 100
            return hash_val < percentage

        return flag.get("default", default)

# Usage
flag_service = FeatureFlagService(flag_store=redis_client)

def create_order(request):
    if flag_service.is_enabled("new_checkout_flow", request.user_id):
        return new_checkout_handler(request)  # New flow (20% users)
    return existing_checkout_handler(request)  # Stable flow
```

```
Feature Flag Lifecycle:
Create → Dark Launch (0%) → Gradual (5→25→50) → GA (100%) → Remove Flag
                                                               ↑
                                                    Technical debt cleanup
```

---

## ၂၅.၄ GitOps — ArgoCD နှင့် Flux

```
GitOps Flow:
Developer ──push──> Git Repo ──detect──> ArgoCD/Flux ──sync──> K8s Cluster

Principles:
1. Declarative: YAML/Helm ဖြင့် desired state describe
2. Versioned: Git history = deployment history
3. Automated: Git change = auto deployment
4. Self-healing: Drift detect + auto-reconcile
```

### ArgoCD

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-production
  namespace: argocd
spec:
  project: microservices
  source:
    repoURL: https://github.com/myorg/k8s-manifests.git
    targetRevision: main
    path: services/order-service/production
    helm:
      valueFiles:
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "abc123def"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true           # Deleted resources cleanup
      selfHeal: true        # Manual changes revert
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Flux

```yaml
# Flux GitRepository + Kustomization
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: k8s-manifests
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/k8s-manifests.git
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: order-service
  namespace: flux-system
spec:
  interval: 5m
  targetNamespace: production
  sourceRef:
    kind: GitRepository
    name: k8s-manifests
  path: ./services/order-service/production
  prune: true
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: order-service
      namespace: production
```

| Feature | ArgoCD | Flux |
|---|---|---|
| **UI** | Rich Web UI | CLI (Weave GitOps optional) |
| **Rollback** | UI one-click | Git revert |
| **RBAC** | Built-in projects | K8s native |
| **Best For** | UI-first teams | Automation-first teams |

---

## ၂၅.၅ Testing Pyramid

```
Testing Pyramid:
              /\
             / E2E \           5% (Slow, Expensive)
            /────────\
           /Integration\      10% (Medium)
          /──────────────\
         / Contract Tests  \  15% (API compat)
        /────────────────────\
       /     Unit Tests        \  70% (Fast, Cheap)
      /──────────────────────────\
```

```go
// Unit Test — mock dependencies
package order

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

type MockInventoryService struct { mock.Mock }

func (m *MockInventoryService) CheckStock(pid string, qty int) (bool, error) {
    args := m.Called(pid, qty)
    return args.Bool(0), args.Error(1)
}

func TestCreateOrder_Success(t *testing.T) {
    mockInv := new(MockInventoryService)
    mockInv.On("CheckStock", "P001", 2).Return(true, nil)
    svc := NewOrderService(mockInv)

    order, err := svc.CreateOrder(CreateOrderRequest{
        UserID: "U001", ProductID: "P001", Quantity: 2,
    })

    assert.NoError(t, err)
    assert.Equal(t, "PENDING", order.Status)
    mockInv.AssertExpectations(t)
}

func TestCreateOrder_OutOfStock(t *testing.T) {
    mockInv := new(MockInventoryService)
    mockInv.On("CheckStock", "P001", 5).Return(false, nil)
    svc := NewOrderService(mockInv)

    order, err := svc.CreateOrder(CreateOrderRequest{
        UserID: "U001", ProductID: "P001", Quantity: 5,
    })

    assert.Error(t, err)
    assert.Nil(t, order)
    assert.Contains(t, err.Error(), "out of stock")
}
```

```python
# Integration Test — real services
import pytest
import requests

def test_create_and_get_order(setup_test_env):
    resp = requests.post("http://localhost:8080/api/orders", json={
        "user_id": "test-001",
        "items": [{"product_id": "P001", "quantity": 2, "price": 29.99}]
    })
    assert resp.status_code == 201
    order_id = resp.json()["id"]

    get_resp = requests.get(f"http://localhost:8080/api/orders/{order_id}")
    assert get_resp.status_code == 200
    assert get_resp.json()["status"] == "PENDING"
```

---

## ၂၅.၆ Contract Testing (Pact)

```
Contract Testing Flow:
Consumer (Order Svc) ──generates──> Pact File ──publish──> Pact Broker
                                                                │
Provider (Inventory Svc) <──fetch + verify──────────────────────┘
```

```javascript
// Consumer Pact Test — Order Service expectations define
const { Pact } = require("@pact-foundation/pact");

describe("Inventory Service Contract", () => {
  const provider = new Pact({
    consumer: "OrderService",
    provider: "InventoryService",
    port: 1234,
  });

  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());

  it("returns stock for product", async () => {
    await provider.addInteraction({
      state: "product P001 has 50 units",
      uponReceiving: "check stock request",
      withRequest: {
        method: "GET",
        path: "/api/inventory/P001",
      },
      willRespondWith: {
        status: 200,
        body: {
          product_id: "P001",
          available: true,
          quantity: 50,
        },
      },
    });

    const client = new InventoryClient("http://localhost:1234");
    const result = await client.checkStock("P001");
    expect(result.available).toBe(true);
    await provider.verify();
  });
});
```

```python
# Provider Verification — Inventory Service
from pact import Verifier

def test_provider_contract():
    verifier = Verifier(
        provider="InventoryService",
        provider_base_url="http://localhost:8080",
    )
    output, _ = verifier.verify_with_broker(
        broker_url="https://pact-broker.myorg.com",
        publish_version="1.2.0",
        provider_states_setup_url="http://localhost:8080/_pact/setup",
        enable_pending=True,
    )
    assert output == 0
```

```bash
# Can-I-Deploy — deploy safe စစ်ဆေးသည်
pact-broker can-i-deploy \
  --pacticipant OrderService \
  --version $(git rev-parse --short HEAD) \
  --to-environment production

# Output: Computer says yes \o/
```

---

## အဓိက အချက်များ (Key Takeaways)

1. **Independent pipelines** — path-based triggers ဖြင့် affected services ကိုသာ build/deploy ပြုလုပ်သည်

2. **Blue-Green** = instant rollback, **Canary** = progressive rollout with automated analysis

3. **Feature flags** — deployment နှင့် release decouple, percentage rollout, kill switch

4. **GitOps** (ArgoCD/Flux) — Git = single source of truth, automated sync + self-healing

5. **Testing pyramid** — Unit (70%) → Contract (15%) → Integration (10%) → E2E (5%)

6. **Contract testing (Pact)** — consumer-provider compatibility CI pipeline တွင် verify

---

> **နောက်အခန်း Preview:** အခန်း ၂၆ တွင် Multi-Region, Disaster Recovery, Chaos Engineering တို့ကို ဆွေးနွေးသွားမည်ဖြစ်သည်။
