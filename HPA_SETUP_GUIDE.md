# Hướng dẫn thiết lập và kiểm tra Auto Scaling (HPA)

File `hpa.yaml` đã được tạo để cấu hình tự động scale số lượng Pod (Horizontal Pod Autoscaler) cho cả **Client** và **Server**.

## 1. Cấu hình hiện tại (`hpa.yaml`)
- **Ngưỡng CPU (Target CPU Utilization):** 70%
- **Số lượng Pod tối thiểu (Min Replicas):** 1 Pod
- **Số lượng Pod tối đa (Max Replicas):** 3 Pods

*Điều này có nghĩa là khi CPU của pod hiện tại hoạt động quá 70% ngưỡng cho phép (250m CPU limits), Kubernetes sẽ tự động tạo thêm pod mới (tối đa là 3 pod) để san sẻ tải. Khi tải giảm xuống, K8s sẽ tự động thu gọn lại (scale in) về 1 pod ban đầu để tiết kiệm tài nguyên.*

## 2. File này có cần đưa vào `.gitignore` không?
**KHÔNG CẦN VÀ KHÔNG NÊN ĐƯA VÀO `.gitignore`.**

Vì bạn đang sử dụng ArgoCD theo mô hình **GitOps**, nguyên lý hoạt động là ArgoCD sẽ nhìn vào trạng thái file trên Git để áp dụng cấu hình (apply) xuống Cluster. 

File `hpa.yaml` là cấu hình K8s public tiêu chuẩn (không chứa secret như token hay password), nên bắt buộc phải nằm trên Git để ArgoCD có thể "nhìn thấy" và tự động apply nó cho bạn.

Mình đã kiểm tra file `.gitignore` của bạn, file `hpa.yaml` hoàn toàn hợp lệ và có thể commit được.

## 3. Cách triển khai lên hệ thống
Bạn có thể áp dụng theo 1 trong 2 cách sau:

### Cách 1: Qua ArgoCD (Đề xuất)
Vì bạn dùng ArgoCD auto pulling, bạn chỉ cần thực hiện quy trình Git chuẩn:
```bash
git add k8s/hpa.yaml
git commit -m "feat: add HPA for client and server to auto-scale on high CPU"
git push
```
Sau vài phút, ArgoCD sẽ tự động lấy code mới về và tạo HPA trong cluster.

### Cách 2: Apply trực tiếp (Dùng để test thử nhanh)
Nếu bạn muốn K8s nhận ngay lập tức, hãy dùng lệnh sau trên server có `kubectl`:
```bash
kubectl apply -f e:\DevopsDaily\k8s\hpa.yaml
```

## 4. Cách theo dõi hoạt động của HPA
Sau khi apply thành công (bằng ArgoCD hoặc thủ công), bạn có thể gõ lệnh sau để xem trạng thái của HPA:

```bash
kubectl get hpa
```

Kết quả sẽ có dạng tương tự thế này:
```text
NAME                     REFERENCE                       TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
devops-blog-client-hpa   Deployment/devops-blog-client   <unknown>/70%   1         3         1          15s
devops-blog-server-hpa   Deployment/devops-blog-server   5%/70%          1         3         1          15s
```

**Lưu ý:** 
- Cột `TARGETS` hiển thị `[CPU đang dùng thực tế] / [CPU cho phép]`. Ví dụ `5%/70%`.
- Khi vừa mới tạo, trạng thái có thể hiện `<unknown> / 70%`. Hãy đợi khoảng 1-2 phút để Metrics Server thu thập đủ dữ liệu.
- Trong trường hợp K8s cluster của bạn chưa có **Metrics Server** (một thành phần bắt buộc để HPA chạy), phần `<unknown>` sẽ hiện mãi. Khi đó, báo lại để mình hướng dẫn bạn cài Metrics Server nhé (với K3s thì thông thường đã có sẵn).
