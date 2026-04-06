# 📖 Tài liệu Kubernetes — DevOps Blog Project

> **Project:** `blog.thienduong.info` (Next.js) + `api.blog.thienduong.info` (Node.js)  
> **Cluster:** K3s single-node VPS  
> **Tác giả:** ThienDuong2909  
> **Cập nhật lần cuối:** 2026-03-30

---

## 📋 Mục lục

1. [Tổng quan kiến trúc](#1-tổng-quan-kiến-trúc)
2. [Danh sách file & vai trò](#2-danh-sách-file--vai-trò)
3. [Hạ tầng cơ bản — Core Workloads](#3-hạ-tầng-cơ-bản--core-workloads)
   - 3.1 [secret.yaml — Quản lý thông tin nhạy cảm](#31-secretyaml--quản-lý-thông-tin-nhạy-cảm)
   - 3.2 [deployment.yaml — Triển khai ứng dụng](#32-deploymentyaml--triển-khai-ứng-dụng)
   - 3.3 [service.yaml — Địa chỉ nội bộ ổn định](#33-serviceyaml--địa-chỉ-nội-bộ-ổn-định)
   - 3.4 [ingress.yaml — Định tuyến từ Internet](#34-ingressyaml--định-tuyến-từ-internet)
   - 3.5 [traefik-redirect.yaml — Cấu hình Traefik nâng cao](#35-traefik-redirectyaml--cấu-hình-traefik-nâng-cao)
4. [Monitoring Stack — Prometheus & Grafana](#4-monitoring-stack--prometheus--grafana)
   - 4.1 [prometheus-values.yaml — Helm values cho kube-prometheus-stack](#41-prometheus-valuesyaml--helm-values-cho-kube-prometheus-stack)
   - 4.2 [server-podmonitor.yaml — Scrape metrics từ Node.js](#42-server-podmonitoryaml--scrape-metrics-từ-nodejs)
   - 4.3 [alert-rules.yaml — Quy tắc cảnh báo](#43-alert-rulesyaml--quy-tắc-cảnh-báo)
5. [Logging Stack — Loki & Promtail](#5-logging-stack--loki--promtail)
   - 5.1 [loki3-values.yaml — Loki 3.x Single Binary](#51-loki3-valuesyaml--loki-3x-single-binary)
   - 5.2 [promtail-values.yaml — Log shipper](#52-promtail-valuesyaml--log-shipper)
   - 5.3 [loki-values.yaml — Loki Stack (legacy reference)](#53-loki-valuesyaml--loki-stack-legacy-reference)
6. [CI/CD Pipeline — GitOps Flow](#6-cicd-pipeline--gitops-flow)
7. [Hướng dẫn Setup từ đầu](#7-hướng-dẫn-setup-từ-đầu)
   - 7.1 [Cài đặt K3s](#71-cài-đặt-k3s)
   - 7.2 [Cài cert-manager](#72-cài-cert-manager)
   - 7.3 [Deploy Core Workloads](#73-deploy-core-workloads)
   - 7.4 [Cài Monitoring Stack](#74-cài-monitoring-stack)
   - 7.5 [Cài Logging Stack](#75-cài-logging-stack)
8. [Bảng tổng hợp Resource Limits](#8-bảng-tổng-hợp-resource-limits)
9. [Câu lệnh kubectl thường dùng](#9-câu-lệnh-kubectl-thường-dùng)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Tổng quan kiến trúc

### 1.1 Sơ đồ luồng request

```
Internet
   │
   ▼ (HTTPS :443)
┌─────────────────────────────────────────────────┐
│  Traefik Ingress Controller (K3s built-in)      │  ← traefik-redirect.yaml
│  - Termination TLS (cert từ Let's Encrypt)      │
│  - HTTP → HTTPS redirect                        │
│  - Expose Prometheus metrics riêng              │
└─────────────────────────────────────────────────┘
   │                          │
   ▼ blog.thienduong.info     ▼ api.blog.thienduong.info
┌──────────────────┐    ┌───────────────────────┐
│  client-svc :80  │    │  server-svc :80        │  ← service.yaml
└──────────────────┘    └───────────────────────┘
   │                          │
   ▼                          ▼
┌──────────────────┐    ┌───────────────────────┐
│  Pod: client     │    │  Pod: server           │  ← deployment.yaml
│  (Next.js :3000) │    │  (Node.js :3001)       │
└──────────────────┘    └───────────────────────┘
                              │
                              ▼ env vars từ
                        ┌─────────────────┐
                        │  Secret         │  ← secret.yaml
                        │  DATABASE_URL   │
                        │  JWT_*_SECRET   │
                        └─────────────────┘
```

### 1.2 Sơ đồ Observability Stack

```
┌─────────────────────────────────────────────────────────────┐
│                    namespace: monitoring                      │
│                                                              │
│  ┌───────────────┐   scrape    ┌─────────────────────────┐  │
│  │  Prometheus   │ ──────────▶ │  Node.js /metrics        │  │
│  │  (metrics DB) │             │  (prom-client)           │  │
│  └───────┬───────┘             └─────────────────────────┘  │
│          │ query                                             │
│          ▼                      ┌─────────────────────────┐  │
│  ┌───────────────┐  push logs  │  Promtail DaemonSet      │  │
│  │   Grafana     │ ◀────────── │  (đọc /var/log/pods/)    │  │
│  │  (dashboard)  │             └──────────┬──────────────┘  │
│  └───────────────┘                        │ push            │
│          ▲                                ▼                  │
│  ┌───────────────┐             ┌─────────────────────────┐  │
│  │ AlertManager  │             │  Loki (log aggregation)  │  │
│  │  → Slack      │             └─────────────────────────┘  │
│  └───────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 Nguyên tắc thiết kế

| Nguyên tắc | Áp dụng |
|---|---|
| **Zero-downtime deploy** | Rolling Update với `maxUnavailable: 0` |
| **Self-healing** | Liveness/Readiness/Startup Probes |
| **GitOps** | Jenkins CI cập nhật `deployment.yaml`, ArgoCD sync |
| **Secrets separation** | `secret.yaml` không bao giờ commit lên Git |
| **Resource isolation** | Mọi pod đều có `requests` & `limits` |
| **HTTPS-only** | Traefik redirect HTTP → HTTPS, cert-manager tự động issue |
| **Observability** | Metrics (Prometheus) + Logs (Loki) + Alerts (Slack) |

---

## 2. Danh sách file & vai trò

| File | Kubernetes Kind | Namespace | Vai trò |
|---|---|---|---|
| `secret.yaml` | Secret | default | Lưu DB credentials, JWT secrets |
| `deployment.yaml` | Deployment × 2 | default | Định nghĩa pods client + server |
| `service.yaml` | Service × 2 | default | Expose pods trong cluster |
| `ingress.yaml` | Ingress | default | Routing từ internet, TLS |
| `traefik-redirect.yaml` | HelmChartConfig | kube-system | Cấu hình Traefik nâng cao |
| `prometheus-values.yaml` | Helm values | monitoring | kube-prometheus-stack config |
| `server-podmonitor.yaml` | PodMonitor | monitoring | Scrape /metrics từ server |
| `alert-rules.yaml` | PrometheusRule | monitoring | Alerting rules (4 groups) |
| `loki3-values.yaml` | Helm values | monitoring | Loki 3.x Single Binary config |
| `promtail-values.yaml` | Helm values | monitoring | Promtail log shipper config |
| `loki-values.yaml` | Helm values | monitoring | Loki Stack legacy (reference) |

> **Thứ tự apply:** `secret.yaml` → `deployment.yaml` → `service.yaml` → `ingress.yaml`  
> (Secret phải có trước Deployment vì Deployment tham chiếu đến Secret)

---

## 3. Hạ tầng cơ bản — Core Workloads

### 3.1 `secret.yaml` — Quản lý thông tin nhạy cảm

> **Áp dụng thủ công:** `kubectl apply -f secret.yaml`  
> **KHÔNG commit lên Git** — đã có trong `.gitignore`

#### Cấu trúc

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: devops-blog-server-secret
  namespace: default
type: Opaque
stringData:
  DATABASE_URL: "mysql://user:password@host:3306/db"
  JWT_ACCESS_SECRET: "..."
  JWT_REFRESH_SECRET: "..."
```

#### Giải thích chi tiết

**`type: Opaque`** — Loại generic cho arbitrary key-value. Các loại khác:
- `kubernetes.io/tls` — chứa TLS cert/key
- `kubernetes.io/dockerconfigjson` — chứa Docker registry credentials

**`stringData` vs `data`:**

```yaml
# ❌ data — phải base64 encode thủ công, dễ nhầm
data:
  DATABASE_URL: bXlzcWw6Ly91c2VyOnBhc3NAaG9zdC9kYg==

# ✅ stringData — nhập plaintext, K8s tự base64 encode
stringData:
  DATABASE_URL: "mysql://user:pass@host/db"
```

#### Cách Secret được inject vào Pod

Secret `devops-blog-server-secret` được tham chiếu trong [`deployment.yaml`](#32-deploymentyaml--triển-khai-ứng-dụng) thông qua `secretKeyRef`:

```yaml
# deployment.yaml — phần server
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: devops-blog-server-secret  # Tên Secret object
        key: DATABASE_URL                # Key trong Secret
  - name: JWT_ACCESS_SECRET
    valueFrom:
      secretKeyRef:
        name: devops-blog-server-secret
        key: JWT_ACCESS_SECRET
  - name: JWT_REFRESH_SECRET
    valueFrom:
      secretKeyRef:
        name: devops-blog-server-secret
        key: JWT_REFRESH_SECRET
```

> ⚠️ Nếu Secret chưa tồn tại khi Pod khởi động → Pod sẽ ở trạng thái `CreateContainerConfigError`

---

### 3.2 `deployment.yaml` — Triển khai ứng dụng

File này định nghĩa **2 Deployment** (client + server) viết trong cùng 1 file, ngăn cách bằng `---`.

#### Cấu trúc phân cấp K8s

```
Deployment
└── quản lý → ReplicaSet (tự động tạo)
               └── quản lý → Pod
                             └── chứa → Container (Docker image)
```

#### 3.2.1 Rolling Update Strategy (Zero-downtime)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1       # Cho phép tạo thêm 1 pod MỚI trước khi xóa pod cũ
    maxUnavailable: 0 # KHÔNG xóa pod cũ cho đến khi pod mới pass readinessProbe
```

**Timeline deploy mới:**

```
Trạng thái ban đầu: [Pod-old RUNNING]

Bước 1: Tạo pod mới    → [Pod-old RUNNING] [Pod-new STARTING]
Bước 2: Chờ readiness  → [Pod-old RUNNING] [Pod-new READY ✅]
Bước 3: Xóa pod cũ    → [Pod-new RUNNING]

→ Không có downtime vì lúc nào cũng có ít nhất 1 pod sẵn sàng nhận traffic
```

#### 3.2.2 Client Deployment (Next.js)

```yaml
metadata:
  name: devops-blog-client
  namespace: default
spec:
  replicas: 1
  revisionHistoryLimit: 5   # Giữ 5 ReplicaSet cũ để rollback
```

**Image:** `thienduong2909/devops-blog-client:<BUILD_NUMBER>`  
→ Jenkins CI tự động cập nhật tag mỗi build thành công (xem [CI/CD Pipeline](#6-cicd-pipeline--gitops-flow))

**Environment variables:**

```yaml
env:
  - name: HOSTNAME
    value: "0.0.0.0"  # QUAN TRỌNG: bind all interfaces để K8s có thể truy cập
  - name: PORT
    value: "3000"
```

> ⚠️ Mặc định Next.js bind vào `127.0.0.1` (localhost only). Trong container phải là `0.0.0.0`

**Health Probes:**

| Probe | Path | Threshold | Mục đích |
|---|---|---|---|
| `startupProbe` | `GET /` :3000 | 6 lần × 10s = 60s | Cho phép 60s để khởi động |
| `livenessProbe` | `GET /` :3000 | 3 lần, mỗi 15s | Restart nếu app bị đơ |
| `readinessProbe` | `GET /` :3000 | 3 lần, mỗi 10s | Không nhận traffic nếu chưa sẵn sàng |

> 📝 **Todo:** Đổi probe path từ `/` → `/api/health` sau khi image mới có endpoint này

#### 3.2.3 Server Deployment (Node.js + Prisma)

**Environment variables:**

```yaml
env:
  - name: NODE_ENV
    value: "production"  # Tắt debug logs, bật optimizations
  - name: PORT
    value: "3001"
  # DATABASE_URL, JWT_* → lấy từ secret (xem section 3.1)
```

**Health Probes:**

| Probe | Path | Threshold | Mục đích |
|---|---|---|---|
| `startupProbe` | `GET /health` :3001 | 10 lần × 10s = **100s** | Đủ thời gian cho Prisma connect DB |
| `livenessProbe` | `GET /health` :3001 | 3 lần, mỗi 15s | Restart nếu DB mất kết nối |
| `readinessProbe` | `GET /health` :3001 | 3 lần, mỗi 10s | Không route traffic nếu DB chưa kết nối |

**Flow probe khi khởi động:**

```
0s    Pod bắt đầu
10s   startupProbe: GET /health → 503 (DB đang kết nối) → fail 1/10
20s   startupProbe: GET /health → 503 → fail 2/10
30s   startupProbe: GET /health → 200 (DB connected!) → PASS ✅

      livenessProbe & readinessProbe bắt đầu chạy
40s   readinessProbe: 200 → Pod được đánh dấu READY → Ingress bắt đầu route traffic
```

#### 3.2.4 Resource Limits (cả 2 pods)

```yaml
resources:
  requests:         # Tài nguyên GUARANTEED — K8s Scheduler dùng để chọn node
    memory: "256Mi"
    cpu: "250m"     # 250 millicores = 25% của 1 CPU core
  limits:           # Tài nguyên TỐI ĐA
    memory: "512Mi" # Vượt → OOMKilled
    cpu: "500m"     # Vượt → CPU throttled (không kill pod)
```

**Đơn vị CPU:** `1000m = 1 core`, `500m = 0.5 core`, `250m = 0.25 core`

---

### 3.3 `service.yaml` — Địa chỉ nội bộ ổn định

> **Vấn đề:** Pod IP thay đổi mỗi lần restart → Service cung cấp một DNS/IP ổn định cố định

#### 2 Services được định nghĩa

**Client Service:**
```yaml
name: devops-blog-client-svc
selector:
  app: devops-blog-client    # Tìm pods có label này (từ deployment.yaml)
ports:
  - port: 80                 # Service port (Ingress gọi vào đây)
    targetPort: 3000         # Container port (Next.js lắng nghe)
type: ClusterIP              # Chỉ accessible trong nội bộ cluster
```

**Server Service:**
```yaml
name: devops-blog-server-svc
selector:
  app: devops-blog-server
ports:
  - port: 80
    targetPort: 3001         # Node.js container port
type: ClusterIP
```

#### Port mapping

```
Internet → Traefik :443 → Ingress → Service :80 → Pod :3000/:3001
                                       ↑ port        ↑ targetPort
```

#### DNS nội bộ tự động

K8s tự tạo DNS records:
```
devops-blog-client-svc.default.svc.cluster.local → ClusterIP
devops-blog-server-svc.default.svc.cluster.local → ClusterIP
```

Prometheus dùng DNS này để scrape server: `devops-blog-server-svc.default.svc.cluster.local:80`  
(xem [`prometheus-values.yaml`](#41-prometheus-valuesyaml--helm-values-cho-kube-prometheus-stack))

#### Tại sao dùng `ClusterIP` thay vì `NodePort`/`LoadBalancer`?

| Type | Dùng khi |
|---|---|
| `ClusterIP` | ✅ Dùng với Ingress Controller — đây là trường hợp này |
| `NodePort` | Expose tạm thời để debug, không qua Ingress |
| `LoadBalancer` | Cloud environment (AWS ELB, GCP LB) |

---

### 3.4 `ingress.yaml` — Định tuyến từ Internet

> Ingress là "bộ định tuyến" nhận request từ internet, phân tích domain, forward đến đúng [`service.yaml`](#33-serviceyaml--địa-chỉ-nội-bộ-ổn-định)

#### Annotations — Chỉ dẫn cho Traefik & cert-manager

```yaml
annotations:
  cert-manager.io/cluster-issuer: letsencrypt-prod
  # ↑ Yêu cầu cert-manager tự động issue TLS cert từ Let's Encrypt

  traefik.ingress.kubernetes.io/router.entrypoints: websecure
  # ↑ Chỉ chấp nhận kết nối HTTPS (port 443), từ chối HTTP
```

#### Routing Rules

```yaml
rules:
  # Frontend: blog.thienduong.info → client service
  - host: "blog.thienduong.info"
    http:
      paths:
        - path: /
          pathType: Prefix       # Match / và mọi path bắt đầu bằng /
          backend:
            service:
              name: devops-blog-client-svc  # Tên trong service.yaml
              port:
                number: 80

  # Backend API: api.blog.thienduong.info → server service
  - host: "api.blog.thienduong.info"
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: devops-blog-server-svc
              port:
                number: 80
```

#### TLS Configuration

```yaml
tls:
  - hosts:
      - blog.thienduong.info
      - api.blog.thienduong.info
    secretName: devops-blog-tls  # Secret sẽ chứa TLS cert & private key
```

#### Luồng tự động issue TLS (cert-manager)

```
1. kubectl apply -f ingress.yaml
       ↓
2. cert-manager phát hiện annotation "letsencrypt-prod"
       ↓
3. cert-manager tạo Certificate object
       ↓
4. cert-manager gọi Let's Encrypt ACME API
       ↓
5. Let's Encrypt gửi HTTP-01 challenge
       ↓
6. Traefik expose challenge endpoint tạm thời
       ↓
7. Let's Encrypt verify domain thành công
       ↓
8. cert-manager nhận cert, lưu vào Secret "devops-blog-tls"
       ↓
9. Traefik đọc Secret → phục vụ HTTPS ✅
       ↓
10. Tự động renew 30 ngày trước khi hết hạn
```

---

### 3.5 `traefik-redirect.yaml` — Cấu hình Traefik nâng cao

> Dùng `HelmChartConfig` (đặc thù K3s) để override Helm values của Traefik mà không cần cài lại

#### HTTP → HTTPS Redirect

```yaml
ports:
  web:
    redirections:
      entryPoint:
        to: websecure
        scheme: https
        permanent: true   # 301 redirect (SEO friendly)
```

#### Prometheus Metrics Endpoint

```yaml
ports:
  metrics:
    port: 9100
    expose:
      default: false      # KHÔNG expose ra ngoài cluster (chỉ trong nội bộ)

metrics:
  prometheus:
    entryPoint: metrics
    addEntryPointsLabels: true   # Label theo entrypoint (web/websecure)
    addRoutersLabels: true       # Label theo router (per-domain)
    addServicesLabels: true      # Label theo service
```

Traefik metrics được Prometheus scrape tự động qua `kube-prometheus-stack`'s ServiceMonitor.

#### Access Logs (JSON format → Promtail)

```yaml
logs:
  access:
    enabled: true
    format: json          # JSON để Promtail parse labels dễ dàng
    fields:
      defaultMode: keep   # Giữ tất cả fields
      headers:
        names:
          User-Agent: keep       # Browser info cho analytics
          Authorization: redact  # Ẩn token (bảo mật)
          Cookie: redact         # Ẩn cookie (GDPR compliance)
```

Traefik ghi JSON access logs ra STDOUT → containerd lưu vào `/var/log/pods/` → [Promtail](#52-promtail-valuesyaml--log-shipper) scrape và parse.

---

## 4. Monitoring Stack — Prometheus & Grafana

### 4.1 `prometheus-values.yaml` — Helm values cho kube-prometheus-stack

> **Cài đặt:**
> ```bash
> helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
>   --namespace monitoring --create-namespace \
>   --values prometheus-values.yaml --wait
> ```

Bộ `kube-prometheus-stack` bao gồm: Prometheus + Grafana + AlertManager + Node Exporter + kube-state-metrics.

#### AlertManager — Gửi alert về Slack

```yaml
alertmanager:
  config:
    global:
      slack_api_url: "https://hooks.slack.com/services/..."
    route:
      receiver: "slack-devops"
      group_by: ["alertname", "severity", "namespace"]
      group_wait: 30s        # Chờ 30s gom chung alerts cùng group
      group_interval: 5m     # Gửi lại sau 5m nếu còn alert mới trong group
      repeat_interval: 4h    # Gửi lại sau 4h nếu alert vẫn còn

      routes:
        - matchers: [severity = info]    → null (suppress)
        - matchers: [severity = none]    → null (suppress)
        - matchers: [alertname = Watchdog] → null (heartbeat)
        - matchers: [severity = critical]  → slack-devops (group_wait: 10s, repeat: 1h)
        - matchers: [severity = warning]   → slack-devops (default)
```

**Slack message format:** Tự động hiển thị Alert name, Severity, Description, thời gian bắt đầu, pod name, link đến Grafana.

#### Grafana

```yaml
grafana:
  adminPassword: "..."   # Đổi password này sau khi deploy!
  persistence:
    enabled: true
    size: 2Gi            # Lưu dashboards & config vào PVC
  resources:
    requests: { cpu: 100m, memory: 128Mi }
    limits:   { cpu: 300m, memory: 256Mi }
```

#### Prometheus

```yaml
prometheus:
  prometheusSpec:
    retention: 7d        # Giữ data 7 ngày

    # Cho phép pick up ServiceMonitor/PodMonitor từ MỌI namespace
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false

    # Scrape Node.js server metrics
    additionalScrapeConfigs:
      - job_name: "devops-blog-server"
        static_configs:
          - targets: ["devops-blog-server-svc.default.svc.cluster.local:80"]
        metrics_path: /metrics
        scrape_interval: 15s
```

**`SelectorNilUsesHelmValues: false`** — Một cài đặt quan trọng: mặc định Prometheus chỉ pick up các ServiceMonitor có cùng Helm release labels. Khi set `false`, nó sẽ pick up **tất cả** ServiceMonitor/PodMonitor trong cluster, bao gồm [`server-podmonitor.yaml`](#42-server-podmonitoryaml--scrape-metrics-từ-nodejs).

#### Tắt components không dùng (K3s không có etcd external)

```yaml
kubeEtcd:            { enabled: false }
kubeScheduler:       { enabled: false }
kubeControllerManager: { enabled: false }
```

---

### 4.2 `server-podmonitor.yaml` — Scrape metrics từ Node.js

> PodMonitor định nghĩa cách Prometheus scrape metrics từ server pod

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: devops-blog-server
  namespace: monitoring          # Phải cùng namespace với Prometheus
  labels:
    release: kube-prometheus-stack  # Label để Prometheus operator pick up
spec:
  namespaceSelector:
    matchNames:
      - default                  # Namespace của server pod

  selector:
    matchLabels:
      app: devops-blog-server    # Match label trong deployment.yaml

  podMetricsEndpoints:
    - port: http                 # Tên port trong pod spec (service.yaml)
      path: /metrics
      interval: 15s              # Scrape mỗi 15 giây
      scrapeTimeout: 10s
```

**Cách hoạt động:**
1. Prometheus Operator phát hiện PodMonitor này (nhờ label `release: kube-prometheus-stack`)
2. Prometheus tìm tất cả pods có `app: devops-blog-server` trong namespace `default`
3. Prometheus scrape `GET /metrics` trên port `http` (3001) mỗi 15 giây
4. Metrics xuất hiện trong Grafana → Explore → Prometheus data source

> Liên quan: Alert rules dùng metrics này như `http_requests_total`, `nodejs_heap_size_used_bytes` (xem [alert-rules.yaml](#43-alert-rulesyaml--quy-tắc-cảnh-báo))

---

### 4.3 `alert-rules.yaml` — Quy tắc cảnh báo

> **Apply:** `kubectl apply -f alert-rules.yaml`

Định nghĩa `PrometheusRule` với 4 nhóm alert, gửi về Slack qua [AlertManager](#alertmanager--gửi-alert-về-slack).

#### Group 1: Service Availability

| Alert | Điều kiện | Severity | Hành động |
|---|---|---|---|
| `PodCrashLooping` *(disabled)* | restart > 3 lần / 15 phút | critical | Kiểm tra logs |
| `PodNotRunning` *(disabled)* | pod phase != Running / 3 phút | critical | `kubectl describe pod` |
| `DeploymentUnavailable` *(disabled)* | available replicas < desired / 2 phút | warning | Check ReplicaSet |

> Các alert này đang được comment out — có thể bật lại khi cần

#### Group 2: API Performance

| Alert | Điều kiện | Severity |
|---|---|---|
| `HighAPIErrorRate` | 4xx + 5xx > 10% trong 5 phút | warning |
| `High5xxErrors` | > 5 lỗi 5xx/phút trong 3 phút | critical |
| `HighP95Latency` | P95 latency > 1000ms trong 5 phút | warning |

**PromQL ví dụ — Error Rate:**
```promql
(
  sum(rate(http_requests_total{job="devops-blog-server", status_code=~"[45].."}[5m]))
  /
  sum(rate(http_requests_total{job="devops-blog-server"}[5m]))
) * 100 > 10
```

#### Group 3: Node.js Runtime

| Alert | Điều kiện | Severity |
|---|---|---|
| `NodeJSHighHeapUsage` | Heap used > 85% total heap / 5 phút | warning |
| `NodeJSHeapCritical` | Heap > 450MB (gần limit 512MB) / 2 phút | critical |
| `NodeJSEventLoopLag` | Event loop lag > 500ms / 3 phút | warning |

> `NodeJSHeapCritical` trigger khi `nodejs_heap_size_used_bytes > 471859200` (450MB)  
> → Khuyến nghị: `kubectl rollout restart deployment/devops-blog-server -n default`

#### Group 4: Infrastructure

| Alert | Điều kiện | Severity |
|---|---|---|
| `NodeHighMemoryUsage` | VPS RAM > 85% / 5 phút | warning |
| `NodeHighCPUUsage` | VPS CPU > 80% / 10 phút | warning |
| `NodeDiskSpaceLow` | Disk còn < 15% / 5 phút | critical |
| `PrometheusTargetDown` | Target `devops-blog-server` down / 2 phút | critical |

**PromQL — CPU Usage:**
```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
```

---

## 5. Logging Stack — Loki & Promtail

### 5.1 `loki3-values.yaml` — Loki 3.x Single Binary

> **Cài đặt:**
> ```bash
> helm install loki grafana/loki \
>   --namespace monitoring \
>   --values loki3-values.yaml
> ```

#### Chọn Single Binary mode

```yaml
deploymentMode: SingleBinary
# Tất cả Loki components (read/write/backend) trong 1 pod
# → Phù hợp VPS đơn node, tiết kiệm RAM đáng kể
```

```yaml
# Tắt các modes không dùng
read:    { replicas: 0 }
write:   { replicas: 0 }
backend: { replicas: 0 }
```

#### Storage — Filesystem thay vì Object Storage

```yaml
loki:
  storage:
    type: filesystem       # Lưu vào PVC (K3s local-path), không cần S3/MinIO

  schemaConfig:
    configs:
      - from: 2024-01-01
        store: tsdb        # TSDB index — schema mới nhất của Loki 3.x
        object_store: filesystem
        schema: v13        # Schema v13 — nên dùng từ 2024+
        index:
          prefix: loki_index_
          period: 24h      # Tạo index mới mỗi ngày
```

#### Retention & Limits

```yaml
limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h  # Từ chối log cũ hơn 7 ngày
  max_cache_freshness_per_query: 10m
  retention_period: 720h             # Giữ logs 30 ngày
```

#### Resources & Storage

```yaml
singleBinary:
  replicas: 1
  persistence:
    enabled: true
    size: 5Gi
    storageClass: local-path         # K3s default storage class

  resources:
    requests: { cpu: 100m, memory: 128Mi }
    limits:   { cpu: 300m, memory: 384Mi }
```

#### Tắt cache (không cần cho VPS nhỏ)

```yaml
chunksCache:  { enabled: false }
resultsCache: { enabled: false }
```

---

### 5.2 `promtail-values.yaml` — Log shipper

> **Cài đặt:**
> ```bash
> helm install promtail grafana/promtail \
>   --namespace monitoring \
>   --values promtail-values.yaml
> ```

Promtail chạy như **DaemonSet** — có 1 pod trên mỗi node K8s, đọc logs của tất cả containers và gửi về Loki.

#### Push đến Loki 3.x

```yaml
clients:
  - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
  # ↑ Dùng gateway URL của Loki 3.x (khác với Loki 2.x dùng loki:3100 trực tiếp)
```

#### Scrape Job 1: Tất cả K8s pod logs

```yaml
- job_name: kubernetes-pods
  pipeline_stages:
    - cri: {}              # Parse containerd log format (K3s dùng containerd)

  kubernetes_sd_configs:
    - role: pod            # Tự động discover tất cả pods

  relabel_configs:
    # Gán labels từ K8s metadata để filter trong Grafana
    - source_labels: [__meta_kubernetes_namespace]   → namespace
    - source_labels: [__meta_kubernetes_pod_name]    → pod
    - source_labels: [__meta_kubernetes_container_name] → container
    - source_labels: [__meta_kubernetes_pod_label_app]  → app
    - source_labels: [__meta_kubernetes_node_name]   → node

    # Đường dẫn log file của container
    - target_label: __path__
      replacement: /var/log/pods/*$pod_uid*/$container/*.log
```

#### Scrape Job 2: Traefik access logs

```yaml
- job_name: traefik-access
  pipeline_stages:
    - cri: {}
    - json:
        expressions:          # Parse JSON fields từ Traefik access log
          ClientAddr: ClientAddr
          RequestMethod: RequestMethod
          RequestPath: RequestPath
          DownstreamStatus: DownstreamStatus
          Duration: Duration
          RequestUserAgent: RequestUserAgent
          RequestHost: RequestHost
    - labels:                 # Promote thành Loki labels để query nhanh
        RequestMethod:
        DownstreamStatus:
        RequestHost:

  relabel_configs:
    # Chỉ lấy Traefik pods trong kube-system
    - action: keep
      regex: traefik          # Label: app.kubernetes.io/name=traefik
    - action: keep
      regex: kube-system      # Namespace filter
```

Nhờ cấu hình này, trong Grafana Explore bạn có thể query:
```logql
{job="traefik-access", DownstreamStatus="500"}
```

---

### 5.3 `loki-values.yaml` — Loki Stack (legacy reference)

> File này dùng `grafana/loki-stack` Helm chart (Loki 2.x + Promtail cài cùng 1 chart)  
> **Hiện không dùng** — đã được thay thế bởi [`loki3-values.yaml`](#51-loki3-valuesyaml--loki-3x-single-binary) + [`promtail-values.yaml`](#52-promtail-valuesyaml--log-shipper) riêng biệt cho Loki 3.x

Lưu lại để tham khảo cấu trúc cũ hoặc rollback nếu cần.

---

## 6. CI/CD Pipeline — GitOps Flow

### 6.1 Tổng quan luồng

```
Developer push code lên GitHub
          ↓
Jenkins phát hiện webhook
          ↓
┌─────────────── Jenkins Pipeline ───────────────────┐
│  1. Checkout Source Code                           │
│  2. npm ci (install dependencies)                  │
│  3. [Server only] npx prisma generate              │
│  4. [Server only] npm run test:coverage            │
│  5. SonarQube Analysis (static code analysis)      │
│  6. Quality Gate (chờ SonarQube OK)                │
│  7. Docker Build & Push → Docker Hub               │
│     image: thienduong2909/devops-blog-{client/server}:BUILD_NUMBER │
│  8. Clone repo Blog_K8S                            │
│  9. sed: cập nhật image tag trong deployment.yaml  │
│  10. git commit & push → Blog_K8S/main             │
└────────────────────────────────────────────────────┘
          ↓
ArgoCD detect thay đổi trong Blog_K8S repo
          ↓
K8s apply deployment.yaml → Rolling Update
          ↓
New pod: pull image mới → pass probes → nhận traffic ✅
Old pod: graceful shutdown
```

### 6.2 Docker Image Strategy

```
Build mới: thienduong2909/devops-blog-client:BUILD_NUMBER (e.g. :59)
           thienduong2909/devops-blog-server:BUILD_NUMBER (e.g. :52)

→ Kết hợp với imagePullPolicy: IfNotPresent:
  Mỗi tag mới → image mới → K8s sẽ pull về → deploy thành công

Tag :latest cũng được push nhưng KHÔNG dùng trong K8s manifest
(dùng :latest với IfNotPresent sẽ không pull image mới!)
```

### 6.3 Race condition protection — Git rebase

Khi cả client lẫn server pipeline chạy song song cùng update `deployment.yaml`:

```
Client pipeline: clone → edit → commit
Server pipeline: clone → edit → commit → push TRƯỚC

Client pipeline cố push: bị reject (remote đã có commit mới)
→ git fetch origin main
→ git rebase origin/main   ← áp dụng commit mới nhất, giữ changes của mình
→ git push origin HEAD:main  ✅
```

### 6.4 Security trong Pipeline

- Credentials (Docker Hub, GitHub token) lưu trong **Jenkins Credentials Store**, không hardcode
- `DOCKER_CONFIG` được set per-workspace → tránh race condition giữa 2 pipelines song song
- `.env.production` được tạo trong build và **xóa sau build** (`post.always`)
- `withCredentials` + `shell interpolation` → Jenkins tự mask token trong console log

---

## 7. Hướng dẫn Setup từ đầu

### 7.1 Cài đặt K3s

```bash
# Cài K3s với Traefik (mặc định có sẵn)
curl -sfL https://get.k3s.io | sh -

# Verify
kubectl get nodes
kubectl get pods -n kube-system

# Config kubectl cho user thường
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

### 7.2 Cài cert-manager

```bash
# Cài cert-manager qua Helm
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Tạo ClusterIssuer cho Let's Encrypt (thay email của bạn)
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: traefik
EOF
```

### 7.3 Deploy Core Workloads

```bash
# Bước 1: Apply Secret (bắt buộc trước)
kubectl apply -f k8s/secret.yaml

# Bước 2: Apply Deployment
kubectl apply -f k8s/deployment.yaml

# Bước 3: Apply Service
kubectl apply -f k8s/service.yaml

# Bước 4: Apply Ingress (cert-manager sẽ tự issue TLS)
kubectl apply -f k8s/ingress.yaml

# Bước 5: Cấu hình Traefik
kubectl apply -f k8s/traefik-redirect.yaml

# Verify
kubectl get pods -n default
kubectl get services -n default
kubectl get ingress -n default
kubectl get certificate -n default    # Xem TLS cert status
```

### 7.4 Cài Monitoring Stack

```bash
# Thêm Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Cài kube-prometheus-stack
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values k8s/prometheus-values.yaml \
  --wait

# Apply PodMonitor và Alert Rules
kubectl apply -f k8s/server-podmonitor.yaml
kubectl apply -f k8s/alert-rules.yaml

# Verify
kubectl get pods -n monitoring
kubectl get prometheusrule -n monitoring
kubectl get podmonitor -n monitoring

# Truy cập Grafana (port-forward để test)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Sau đó mở http://localhost:3000 (admin/password từ prometheus-values.yaml)
```

### 7.5 Cài Logging Stack

```bash
# Thêm Grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Cài Loki 3.x
helm install loki grafana/loki \
  --namespace monitoring \
  --values k8s/loki3-values.yaml \
  --wait

# Cài Promtail
helm install promtail grafana/promtail \
  --namespace monitoring \
  --values k8s/promtail-values.yaml

# Verify
kubectl get pods -n monitoring | grep -E "loki|promtail"

# Thêm Loki data source vào Grafana
# Grafana → Connections → Data Sources → Add → Loki
# URL: http://loki-gateway.monitoring.svc.cluster.local
```

---

## 8. Bảng tổng hợp Resource Limits

| Component | CPU Request | CPU Limit | RAM Request | RAM Limit |
|---|---|---|---|---|
| **devops-blog-client** | 250m | 500m | 256Mi | 512Mi |
| **devops-blog-server** | 250m | 500m | 256Mi | 512Mi |
| **Prometheus** | 200m | 800m | 512Mi | 1Gi |
| **Grafana** | 100m | 300m | 128Mi | 256Mi |
| **AlertManager** | 50m | 100m | 64Mi | 128Mi |
| **Loki** | 100m | 300m | 128Mi | 384Mi |
| **Promtail** (per node) | 50m | 100m | 64Mi | 128Mi |

> **Tổng ước tính:** ~1.1 CPU cores + ~2.7GB RAM  
> Phù hợp VPS 4GB RAM, 2 vCPU trở lên

---

## 9. Câu lệnh kubectl thường dùng

### Kiểm tra trạng thái

```bash
# Xem tất cả pods
kubectl get pods -A

# Xem logs pod
kubectl logs -n default deployment/devops-blog-server --tail=100 -f

# Xem events (debug khi pod không start)
kubectl describe pod -n default <pod-name>

# Xem resource usage
kubectl top pods -n default
kubectl top nodes
```

### Deploy & Rollback

```bash
# Rollout restart (restart all pods)
kubectl rollout restart deployment/devops-blog-server -n default
kubectl rollout restart deployment/devops-blog-client -n default

# Xem rollout history
kubectl rollout history deployment/devops-blog-server -n default

# Rollback về version trước
kubectl rollout undo deployment/devops-blog-server -n default

# Rollback về version cụ thể
kubectl rollout undo deployment/devops-blog-server -n default --to-revision=3
```

### Secrets

```bash
# Xem danh sách secrets (không hiện value)
kubectl get secret -n default

# Decode 1 key trong secret
kubectl get secret devops-blog-server-secret \
  -o jsonpath='{.data.DATABASE_URL}' | base64 -d

# Cập nhật giá trị trong secret
kubectl create secret generic devops-blog-server-secret \
  --from-literal=DATABASE_URL="new-url" \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Monitoring

```bash
# Xem alert rules
kubectl get prometheusrule -n monitoring

# Xem targets đang scrape
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# → http://localhost:9090/targets

# Xem AlertManager
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
# → http://localhost:9093
```

### Logs với Loki

```bash
# Query logs qua Grafana → Explore → Loki data source
# LogQL examples:
{namespace="default", app="devops-blog-server"}
{job="traefik-access", DownstreamStatus="500"}
{namespace="default"} |= "ERROR"
```

---

## 10. Troubleshooting

### Pod không start — `CreateContainerConfigError`
```bash
kubectl describe pod <pod-name> -n default
# → Tìm message: "secret devops-blog-server-secret not found"
# Fix: kubectl apply -f k8s/secret.yaml
```

### Pod bị `OOMKilled`
```bash
kubectl describe pod <pod-name> -n default
# → "OOMKilled" trong Reason
# Fix: tăng memory limit trong deployment.yaml
# Xem xu hướng heap trong Grafana → Node.js Runtime dashboard
```

### TLS cert không được issue
```bash
kubectl describe certificate devops-blog-tls -n default
kubectl describe certificaterequest -n default
kubectl logs -n cert-manager deployment/cert-manager
# Thường do: DNS chưa trỏ đúng, domain chưa accessible
```

### Prometheus không scrape được server
```bash
# Kiểm tra target trong Prometheus UI
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# → http://localhost:9090/targets → tìm "devops-blog-server"
# Kiểm tra PodMonitor
kubectl get podmonitor -n monitoring devops-blog-server -o yaml
# Kiểm tra /metrics endpoint
kubectl exec -n default deployment/devops-blog-server -- curl localhost:3001/metrics
```

### Traefik không redirect HTTP → HTTPS
```bash
kubectl get helmchartconfig traefik -n kube-system
kubectl rollout restart -n kube-system deployment/traefik
```

### Logs không xuất hiện trong Loki
```bash
# Kiểm tra Promtail
kubectl logs -n monitoring daemonset/promtail --tail=50
# Kiểm tra Promtail đang gửi đến đúng URL Loki
kubectl get configmap -n monitoring promtail -o yaml | grep url
# URL phải là: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
```

---

*Tài liệu được tạo tự động từ codebase thực của project. Cập nhật khi có thay đổi cấu hình.*
