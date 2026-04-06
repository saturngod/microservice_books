# အခန်း ၂၄ — Containers နှင့် Kubernetes

## နိဒါန်း

Microservices များကို production တွင် deploy ပြုလုပ်ရာ၌ containerization နှင့် orchestration သည် မရှိမဖြစ် လိုအပ်သော technology ဖြစ်သည်။ **Docker** သည် application ကို portable container image အဖြစ် package ပြုလုပ်ပေးပြီး **Kubernetes** သည် containers ရာနှင့်ချီ ထောင်နှင့်ချီကို automate ပြုလုပ်ပြီး manage ပေးသည်။ ဤအခန်းတွင် Docker best practices, Kubernetes core concepts (Pod, Deployment, Service, Ingress, ConfigMap), auto-scaling (HPA & VPA), Helm Charts, StatefulSets, နှင့် Namespaces/Multi-Tenancy တို့ကို အသေးစိတ် လေ့လာသွားမည်ဖြစ်သည်။

---

## ၂၄.၁ Docker Best Practices

### Multi-Stage Build

Multi-stage build ဖြင့် final image size ကို minimize ပြုလုပ်နိုင်သည်။ Build dependencies များကို final image တွင် မပါစေဘဲ production artifact သာ ပါစေသည်။

```dockerfile
# Multi-stage build ဥပမာ — Go microservice
# Stage 1: Build stage — Go compiler နှင့် dependencies ပါသည်
FROM golang:1.22-alpine AS builder
WORKDIR /app

# Dependency files ကို copy ပြီး download ပြုလုပ်သည်
# Source code copy မလုပ်ခင် dependencies cache ဖြစ်စေသည်
COPY go.mod go.sum ./
RUN go mod download

# Source code copy ပြီး compile ပြုလုပ်သည်
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /order-service ./cmd/server

# Stage 2: Production stage — minimal base image
# scratch သုံးပါက shell/debugging tools မပါ (distroless ပိုကောင်းသည်)
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /order-service /order-service
# Non-root user ဖြင့် run — security best practice
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/order-service"]
```

```
Image Size Comparison:
┌──────────────────────────┬──────────────┐
│ Base Image               │ Size         │
├──────────────────────────┼──────────────┤
│ golang:1.22              │ ~800 MB      │
│ golang:1.22-alpine       │ ~250 MB      │
│ alpine:3.19              │ ~7 MB        │
│ distroless/static        │ ~2 MB        │
│ scratch                  │ 0 MB         │
├──────────────────────────┼──────────────┤
│ Final (multi-stage+dist) │ ~12 MB       │  ← ဤသို့ သေးငယ်သည်
└──────────────────────────┴──────────────┘
```

### Dockerfile Best Practices — Node.js ဥပမာ

```dockerfile
# Node.js microservice — best practices ပြည့်စုံသော ဥပမာ
FROM node:20-alpine AS builder
WORKDIR /app

# Layer caching optimize — package files ကို source code ထက် အရင် copy
COPY package.json package-lock.json ./
# ci flag ဖြင့် deterministic install — lock file အတိုင်း install
RUN npm ci --only=production

COPY src/ ./src/

# Production stage
FROM node:20-alpine
WORKDIR /app

# Non-root user ဖန်တီးသည် — container ကို root ဖြင့် run မသင့်
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder --chown=appuser:appgroup /app ./

USER appuser

# Health check — Kubernetes liveness probe နှင့် ပေါင်းစပ်အသုံးပြုနိုင်သည်
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

EXPOSE 8080
CMD ["node", "src/index.js"]
```

### .dockerignore

```
# .dockerignore — build context မှ exclude ပြုလုပ်ရမည့် files
node_modules
.git
.gitignore
*.md
docker-compose*.yml
.env
.env.*
tests/
coverage/
*.log
```

### Docker Best Practices Summary

| Practice | ရှင်းလင်းချက် |
|---|---|
| **Multi-stage builds** | Final image size minimize — build tools မပါ |
| **Layer ordering** | မပြောင်းလဲသော layers (dependencies) ကို အပေါ်တွင် ထား — cache hit |
| **.dockerignore** | node_modules, .git, test files များကို build context မှ exclude |
| **Non-root user** | Container ကို root ဖြင့် run ခြင်း ရှောင်ကြဉ် — CVE exploit minimize |
| **HEALTHCHECK** | Container health status Docker/K8s သို့ report |
| **Pin versions** | `node:20.11-alpine` exact version pin — reproducible builds |
| **Minimal base** | Alpine/distroless/scratch — attack surface minimize |
| **No secrets** | Image ထဲတွင် API keys, passwords မထည့် — runtime inject |

---

## ၂၄.၂ Kubernetes Core Concepts

### Kubernetes Architecture Overview

```
Kubernetes Cluster Architecture:
┌──────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE                             │
│  ┌────────────┐ ┌──────────────┐ ┌────────────┐ ┌───────────┐  │
│  │ API Server │ │   etcd       │ │ Scheduler  │ │ Controller│  │
│  │ (kube-api) │ │ (key-value)  │ │            │ │  Manager  │  │
│  └────────────┘ └──────────────┘ └────────────┘ └───────────┘  │
└──────────────────────────────────────────────────────────────────┘
         │                │                │               │
         ▼                ▼                ▼               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     WORKER NODES                                 │
│  ┌─────────────────────────┐  ┌─────────────────────────┐       │
│  │  Node 1                 │  │  Node 2                 │       │
│  │  ┌───────┐ ┌──────────┐ │  │  ┌───────┐ ┌──────────┐│       │
│  │  │kubelet│ │kube-proxy│ │  │  │kubelet│ │kube-proxy││       │
│  │  └───────┘ └──────────┘ │  │  └───────┘ └──────────┘│       │
│  │  ┌─────┐ ┌─────┐       │  │  ┌─────┐ ┌─────┐       │       │
│  │  │Pod A│ │Pod B│       │  │  │Pod C│ │Pod D│       │       │
│  │  └─────┘ └─────┘       │  │  └─────┘ └─────┘       │       │
│  └─────────────────────────┘  └─────────────────────────┘       │
└──────────────────────────────────────────────────────────────────┘

API Server: cluster ၏ frontend — kubectl commands handle
etcd: cluster state store — distributed key-value store
Scheduler: Pod ကို node ပေါ်တွင် schedule ပြုလုပ်သည်
Controller Manager: desired state == actual state ဖြစ်အောင် ထိန်းညှိ
kubelet: node ပေါ်တွင် container runtime manage
kube-proxy: service networking rules manage
```

### Pod

Pod သည် Kubernetes ၏ smallest deployable unit ဖြစ်သည်။ Container တစ်ခု သို့မဟုတ် တစ်ခုထက်ပိုသော containers ပါဝင်ပြီး network namespace နှင့် storage volumes share ပြုလုပ်သည်။

```yaml
# Pod definition ဥပမာ
apiVersion: v1
kind: Pod
metadata:
  name: order-service
  labels:
    app: order-service
    version: v1
spec:
  containers:
    - name: order-service
      image: myregistry/order-service:v1.2.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: "250m"      # 0.25 CPU core request
          memory: "256Mi"  # 256 MiB memory request
        limits:
          cpu: "500m"      # Maximum 0.5 CPU core
          memory: "512Mi"  # Maximum 512 MiB — OOMKill threshold
      # Readiness probe — traffic receive ready ဖြစ်/မဖြစ် စစ်ဆေး
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
      # Liveness probe — container alive ဖြစ်/မဖြစ် စစ်ဆေး
      livenessProbe:
        httpGet:
          path: /health/live
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 20
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: order-config
              key: database-host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: order-secrets
              key: database-password
```

### Deployment

Deployment သည် Pod များကို declarative manage ပြုလုပ်ပြီး rolling update, rollback, scaling support ပြုလုပ်သည်။

```yaml
# Deployment — production-ready configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Update အတွင်း 1 ခု ပိုရှိနိုင်
      maxUnavailable: 0     # Available replicas မလျော့စေ
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v1.2.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      # Pod anti-affinity — pods ကို nodes ပေါ်တွင် spread
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: order-service
                topologyKey: kubernetes.io/hostname
      terminationGracePeriodSeconds: 30
      containers:
        - name: order-service
          image: myregistry/order-service:v1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
          startupProbe:
            httpGet:
              path: /health/live
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
```

### Service

Service သည် Pod group ကို stable network endpoint ပေးခြင်းဖြစ်သည်။

```yaml
# ClusterIP Service — internal communication
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
---
# LoadBalancer Service — external access
apiVersion: v1
kind: Service
metadata:
  name: order-service-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: order-service
  ports:
    - port: 443
      targetPort: 8080
```

```
Service Types:
┌───────────────┬──────────────────────────────────────────┐
│ Service Type  │ Access Scope                              │
├───────────────┼──────────────────────────────────────────┤
│ ClusterIP     │ Cluster internal only (default)           │
│ NodePort      │ Node IP + high port (30000-32767)         │
│ LoadBalancer  │ Cloud provider external LB                │
│ ExternalName  │ DNS CNAME redirect to external service    │
└───────────────┴──────────────────────────────────────────┘
```

### Ingress

```yaml
# Ingress — path-based routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.myapp.com
      secretName: api-tls-cert
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
          - path: /api/products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 80
```

### ConfigMap & Secret

```yaml
# ConfigMap — non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-config
  namespace: production
data:
  database-host: "postgres-primary.production.svc.cluster.local"
  database-port: "5432"
  database-name: "orders_db"
  log-level: "info"
  application.yaml: |
    server:
      port: 8080
    cache:
      ttl: 300
      max-size: 1000
---
# Secret — sensitive data (production တွင် sealed-secrets/vault သုံးသင့်)
apiVersion: v1
kind: Secret
metadata:
  name: order-secrets
  namespace: production
type: Opaque
data:
  database-password: bXlwYXNzd29yZA==
  api-key: c2VjcmV0LWFwaS1rZXk=
```

---

## ၂၄.၃ Horizontal Pod Autoscaler (HPA) & Vertical Pod Autoscaler (VPA)

### HPA — Horizontal Scaling

```yaml
# HPA — CPU/memory/custom metrics based autoscaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

```
HPA Scaling Formula:
desiredReplicas = ceil(currentReplicas * (currentMetric / targetMetric))

ဥပမာ: currentReplicas=3, CPU=85%, target=70%
desiredReplicas = ceil(3 * 85/70) = ceil(3.64) = 4 pods
```

### VPA — Vertical Scaling

```yaml
# VPA — automatic resource right-sizing
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: "Off"     # Recommendation only (safe mode)
  resourcePolicy:
    containerPolicies:
      - containerName: order-service
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "2"
          memory: "2Gi"
```

| Aspect | HPA | VPA |
|---|---|---|
| **Direction** | Horizontal (Pod count) | Vertical (Pod resources) |
| **Best For** | Stateless, traffic spikes | Right-sizing, cost optimization |
| **Risk** | Over-provisioning | Pod restart |

---

## ၂၄.၄ Helm Charts

```
Helm Chart Structure:
order-service-chart/
├── Chart.yaml
├── values.yaml
├── values-staging.yaml
├── values-production.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   └── configmap.yaml
└── charts/
```

```yaml
# values.yaml — default values
replicaCount: 3

image:
  repository: myregistry/order-service
  tag: "v1.2.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 50
  targetCPUUtilization: 70
```

```yaml
# templates/deployment.yaml — Go template syntax
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```bash
# Helm commands
helm install order-service ./order-service-chart -f values-production.yaml -n production
helm upgrade order-service ./order-service-chart --set image.tag="v1.3.0" -n production
helm rollback order-service 1 -n production
helm history order-service -n production
```

---

## ၂၄.၅ StatefulSets

```
Deployment vs StatefulSet:
┌────────────────────────────┬────────────────────────────┐
│  Deployment (Stateless)    │  StatefulSet (Stateful)    │
│  Pod names: random         │  Pod names: ordered        │
│  web-abc123, web-def456    │  db-0, db-1, db-2          │
│  Scale: parallel           │  Scale: sequential         │
│  Storage: ephemeral        │  Storage: per-pod PVC      │
│  Delete: any order         │  Delete: reverse order     │
└────────────────────────────┴────────────────────────────┘
```

```yaml
# StatefulSet — PostgreSQL cluster
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless
  replicas: 3
  podManagementPolicy: OrderedReady
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3-encrypted
        resources:
          requests:
            storage: 100Gi
---
# Headless Service — stable DNS for StatefulSet pods
# DNS: postgres-0.postgres-headless.production.svc.cluster.local
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

---

## ၂၄.၆ Namespaces & Multi-Tenancy

### Namespace + ResourceQuota + LimitRange

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-orders
  labels:
    team: orders
    environment: production
---
# ResourceQuota — namespace level resource limits
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-orders-quota
  namespace: team-orders
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"
    services: "20"
---
# LimitRange — container level defaults
apiVersion: v1
kind: LimitRange
metadata:
  name: team-orders-limits
  namespace: team-orders
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "250m"
        memory: "256Mi"
      max:
        cpu: "4"
        memory: "8Gi"
```

### Network Policies

```yaml
# Namespace isolation — authorized traffic only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: team-orders
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}           # Same namespace
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx   # Ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring      # Prometheus
```

### Multi-Tenancy Models

```
┌────────────────────────────────────────────────────────────────┐
│  Model 1: Namespace per Team (Soft Isolation)                 │
│  Same cluster, RBAC + ResourceQuota + NetworkPolicy           │
│  Cost: Low — Isolation: Medium                                │
├────────────────────────────────────────────────────────────────┤
│  Model 2: Cluster per Environment (Hard Isolation)            │
│  Separate clusters (dev/staging/prod)                         │
│  Cost: High — Isolation: High                                 │
├────────────────────────────────────────────────────────────────┤
│  Model 3: Virtual Cluster (vCluster)                          │
│  Virtual API server per tenant on shared host cluster         │
│  Cost: Medium — Isolation: High                               │
└────────────────────────────────────────────────────────────────┘
```

### RBAC

```yaml
# Role — namespace scoped permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-orders-developer
  namespace: team-orders
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]        # Read only
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-orders-dev-binding
  namespace: team-orders
subjects:
  - kind: Group
    name: "team-orders-devs"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-orders-developer
  apiGroup: rbac.authorization.k8s.io
```

---

## အဓိက အချက်များ (Key Takeaways)

1. **Docker best practices** — multi-stage builds, minimal base images (distroless), non-root user, layer caching optimize ပြုလုပ်ခြင်းဖြင့် secure/lightweight images ရရှိသည်

2. **Kubernetes core objects** — Pod, Deployment, Service, Ingress, ConfigMap/Secret ကို production deployment အတွက် ပေါင်းစပ်အသုံးပြုသည်

3. **HPA** သည် Pod count auto-scale ပြုလုပ်ပြီး **VPA** သည် Pod resources right-size ပြုလုပ်သည်

4. **Helm Charts** ဖြင့် manifests ကို template ပြုလုပ်ပြီး environment-specific deployment ပြုလုပ်နိုင်သည်

5. **StatefulSets** သည် databases ကဲ့သို့ stateful workloads အတွက် stable identity + persistent storage guarantee ပေးသည်

6. **Namespaces** + ResourceQuota + NetworkPolicy + RBAC ဖြင့် multi-tenancy isolation ပြုလုပ်နိုင်သည်

---

> **နောက်အခန်း Preview:** အခန်း ၂၅ တွင် CI/CD for Microservices — independent pipelines, deployment strategies, feature flags, GitOps, testing pyramid, contract testing တို့ကို ဆွေးနွေးသွားမည်ဖြစ်သည်။
