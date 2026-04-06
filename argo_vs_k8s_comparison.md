# 🔍 So sánh ArgoCD UI vs Cấu hình K8s

## Tổng quan

So sánh các resource hiển thị trên **ArgoCD** (từ ảnh chụp) với các file YAML trong `e:\DevopsDaily\k8s`.

- **App ArgoCD**: `devops-blog`
- **Trạng thái**: ✅ Healthy | ✅ Synced to HEAD (`871f5e5`)
- **Auto sync**: Enabled
- **Last Sync**: Tue Mar 31 2026 09:40:44 (GMT+07:00) — **Sync OK**

---

## Bảng so sánh tổng hợp

| # | Resource trên ArgoCD | Kind | File YAML tương ứng | Khớp? |
|---|---------------------|------|---------------------|-------|
| 1 | `devops-blog-client-svc` | Service | `service.yaml` (line 1-15) | ✅ Khớp |
| 2 | `devops-blog-server-svc` | Service | `service.yaml` (line 16-32) | ✅ Khớp |
| 3 | `devops-blog-client` | Deployment | `deployment.yaml` (line 1-71) | ✅ Khớp |
| 4 | `devops-blog-server` | Deployment | `deployment.yaml` (line 72-161) | ✅ Khớp |
| 5 | `devops-blog-server` | PodMonitor | `server-podmonitor.yaml` | ✅ Khớp |
| 6 | `devops-blog-alerts` | PrometheusRule | `alert-rules.yaml` | ✅ Khớp |
| 7 | `devops-blog-ingress` | Ingress | `ingress.yaml` | ✅ Khớp |
| 8 | `devops-blog-tls` | Certificate | `ingress.yaml` (cert-manager auto-generate từ tls spec) | ✅ Khớp |

---

## Phân tích chi tiết từng Resource

### 1. Services (2 resources)

**ArgoCD hiển thị:**
- `devops-blog-client-svc` (svc)
- `devops-blog-server-svc` (svc)

**File `service.yaml`:**
- `devops-blog-client-svc` → ClusterIP, port 80 → targetPort 3000 ✅
- `devops-blog-server-svc` → ClusterIP, port 80 → targetPort 3001 ✅

> [!TIP]
> Cả 2 service đều dùng ClusterIP (internal only), traffic bên ngoài đi qua Ingress → đúng kiến trúc.

---

### 2. Deployments (2 resources)

**ArgoCD hiển thị:**
- `devops-blog-client` (deploy) — hiển thị `rev:15` → tương ứng `revisionHistoryLimit: 5`
- `devops-blog-server` (deploy) — hiển thị `rev:20` → tương ứng `revisionHistoryLimit: 5`

**File `deployment.yaml`:**
- `devops-blog-client`: replicas=1, image=`thienduong2909/devops-blog-client:59` ✅
- `devops-blog-server`: replicas=1, image=`thienduong2909/devops-blog-server:52` ✅

**ReplicaSets trên ArgoCD** (con của deployment):

| Deployment | ReplicaSets hiển thị | Ý nghĩa |
|------------|---------------------|----------|
| `devops-blog-client` | `ec747obb5` (active), `efe6b557df`, `e7b4b0bt5f`, `5dcc9bb98...` | Nhiều revision cũ được giữ lại (tối đa 5) |
| `devops-blog-server` | `777f5b67...` (active), `dbef4d0be`, `d74b8c895`, `bc8bc98f...`, `557c0c75...` | Nhiều revision cũ được giữ lại (tối đa 5) |

> [!NOTE]
> ArgoCD hiển thị nhiều ReplicaSets cũ do `revisionHistoryLimit: 5` trong deployment spec. Đây là hành vi bình thường — K8s giữ lại RS cũ để hỗ trợ rollback.

---

### 3. PodMonitor

**ArgoCD hiển thị:**
- `devops-blog-server` (podmonitor) — namespace: monitoring

**File `server-podmonitor.yaml`:**
- name: `devops-blog-server`, namespace: `monitoring` ✅
- Label: `release: kube-prometheus-stack` ✅
- Scrape endpoint: `/metrics` trên port `http` (3001), interval `15s` ✅

---

### 4. PrometheusRule

**ArgoCD hiển thị:**
- `devops-blog-alerts` (prometheusrule) — hiển thị `25 days` old

**File `alert-rules.yaml`:**
- name: `devops-blog-alerts`, namespace: `monitoring` ✅
- 4 groups: service.availability, api.performance, nodejs.runtime, infrastructure ✅
- Nhiều rules trong group 1 đang bị **comment out** (PodCrashLooping, PodNotRunning, DeploymentUnavailable)

> [!IMPORTANT]
> Một số alert rules trong group `service.availability` đang bị **comment out**. Nếu cần monitor pod crash/unavailable, hãy bỏ comment các rules đó.

---

### 5. Ingress

**ArgoCD hiển thị:**
- `devops-blog-ingress` (ing)

**File `ingress.yaml`:**
- 2 rules: `blog.thienduong.info` → client-svc, `api.blog.thienduong.info` → server-svc ✅
- TLS enabled với cert-manager (`letsencrypt-prod`) ✅
- Secret name: `devops-blog-tls` ✅

---

### 6. Certificate (TLS)

**ArgoCD hiển thị:**
- `devops-blog-tls` (certificate) — con của Ingress

**File `ingress.yaml` (tls section):**
```yaml
tls:
  - hosts:
      - blog.thienduong.info
      - api.blog.thienduong.info
    secretName: devops-blog-tls
```

> [!NOTE]
> Certificate `devops-blog-tls` được **cert-manager tự động tạo** từ annotation `cert-manager.io/cluster-issuer: letsencrypt-prod` trong Ingress. Không cần file YAML riêng — đây là hành vi đúng.

---

### 7. Secret (không hiển thị trên ArgoCD tree)

**File `secret.yaml`:**
- name: `devops-blog-server-secret`, namespace: `default`
- Được **exclude khỏi Git** (`.gitignore`) → apply thủ công

> [!WARNING]
> Secret **không hiển thị** trên ArgoCD tree vì nó nằm trong `.gitignore` và được apply thủ công. Đây là đúng — secret không nên commit lên Git.

---

## Các file không liên quan trực tiếp đến ArgoCD app

| File | Mục đích | Hiển thị trên ArgoCD? |
|------|----------|----------------------|
| `prometheus-values.yaml` | Helm values cho kube-prometheus-stack | ❌ (Helm release riêng) |
| `loki-values.yaml` | Helm values cho Loki | ❌ (Helm release riêng) |
| `loki3-values.yaml` | Helm values cho Loki v3 | ❌ (Helm release riêng) |
| `promtail-values.yaml` | Helm values cho Promtail | ❌ (Helm release riêng) |
| `traefik-redirect.yaml` | Traefik HTTP→HTTPS redirect middleware | ❌ (Apply riêng) |
| `secret.yaml` | K8s Secret (trong .gitignore) | ❌ (Apply thủ công) |

> [!NOTE]
> Các file Helm values (`*-values.yaml`) và `traefik-redirect.yaml` không thuộc ArgoCD app `devops-blog`. Chúng được deploy qua Helm hoặc apply riêng biệt.

---

## 🏁 Kết luận

### ✅ TẤT CẢ CẤU HÌNH KHỚP HOÀN TOÀN

Tất cả **8 resources** hiển thị trên ArgoCD đều **khớp chính xác** với các file YAML trong thư mục `k8s/`:

1. ✅ 2 Services (`client-svc`, `server-svc`)
2. ✅ 2 Deployments (`client`, `server`) với đúng image tags
3. ✅ 1 PodMonitor (`devops-blog-server`)
4. ✅ 1 PrometheusRule (`devops-blog-alerts`)
5. ✅ 1 Ingress (`devops-blog-ingress`)
6. ✅ 1 Certificate (`devops-blog-tls` — auto-generated bởi cert-manager)

**Trạng thái ArgoCD**: App **Healthy** + **Synced** xác nhận rằng cấu hình trên cluster đang đồng bộ hoàn toàn với Git repository.

> [!TIP]
> Lưu ý nhỏ: Một số alert rules trong `alert-rules.yaml` đang bị comment out (PodCrashLooping, PodNotRunning, DeploymentUnavailable). Cân nhắc bật lại nếu cần giám sát tốt hơn.
