# Deep dive — Metrics, Events & Watch

> Cấp độ mục tiêu: **Junior → Middle**. Toàn bộ nội dung *(bổ sung từ nghiên
> cứu ngoài video)*; nền tảng logs/metrics/traces đã có *(nhắc lại Bài 17)*,
> phương pháp troubleshooting tổng quát *(nhắc lại Bài 16 Phần 1)*.

## ① Concept

**`kubectl top`** đọc từ **Metrics API** (`metrics.k8s.io`) — do addon
**metrics-server** cung cấp, không phải từ etcd như `kubectl get`. Điểm
cần nhớ:

- metrics-server scrape kubelet khoảng **15 giây/lần**, giữ **in-memory**
  → không có lịch sử. Muốn xem "CPU tuần trước" → cần Prometheus/VictoriaMetrics.
- Một số môi trường không cài metrics-server → `kubectl top` trả lỗi
  `error: Metrics API not available`.

**Events** là bản ghi **control-plane thay đổi trạng thái gì** trong
namespace (default TTL 1 giờ — do kube-controller-manager quản lý
**Không nằm trong đề CKAD**, nhưng giải thích vì sao "sự cố hôm qua"
không còn event). `kubectl describe` hiển thị events kèm object; `kubectl
get events` cho toàn cảnh theo thời gian.

**Watch** (`-w`) giữ kết nối tới API server và stream thay đổi — dùng để
quan sát một object "chuyển trạng thái lúc nào" thay vì gõ lại lệnh.

So sánh 4 nguồn quan sát:

| Nguồn | Trả lời câu hỏi | Công cụ |
|---|---|---|
| Metrics | "Bao nhiêu?" — số đo theo thời gian | `kubectl top`, Prometheus |
| Logs | "Ứng dụng nói gì?" | `kubectl logs` |
| Events | "Control-plane làm gì?" | `kubectl get events`, `describe` |
| Traces | "Request đi qua đâu?" | distributed tracing (ngoài phạm vi) |

## ② Configuration

`kubectl top` không cần file YAML — cấu hình nằm ở metrics-server. Trên
minikube: `minikube addons enable metrics-server`.

## ③ Command (kubectl)

```bash
kubectl top pods                          # CPU/memory hiện tại, pod này
kubectl top pods -A                       # mọi namespace
kubectl top pods --sort-by=memory         # tìm pod ăn RAM nhất
kubectl top pods --containers             # tách theo container trong pod
kubectl top nodes                         # tổng tài nguyên từng node

kubectl get events -A \
  --sort-by=.metadata.creationTimestamp   # events cũ → mới
kubectl get events -w                     # stream event mới
kubectl get pods -w                       # theo dõi trạng thái pod
```

Bảng tra cứu **reason thường gặp** khi troubleshooting:

| Reason trong event | STATUS thấy ở pod | Nghĩa |
|---|---|---|
| `FailedScheduling` | Pending | Không đủ resource/node thỏa constraint |
| `FailedMount` | ContainerCreating | Volume/PVC hoặc ConfigMap không tồn tại |
| `Failed` + `BackOff` | ImagePullBackOff | Pull image sai tên/tag/registry |
| `BackOff` | CrashLoopBackOff | Container crash liên tục |
| `Unhealthy` | Running + restarts | Probe fail — xem logs |
| `Evicted` | Evicted | Node thiếu memory/disk → kubelet đuổi pod |

## ④ Production scenarios

**Kịch bản chuẩn: node báo memory pressure.**
1. `kubectl top pods -A --sort-by=memory` → tìm pod dùng RAM nhiều nhất.
2. `kubectl get events -A --sort-by=.metadata.creationTimestamp | tail` →
   xác nhận `Evicted`/`FailedScheduling` gần nhất.
3. `kubectl top nodes` → node nào căng → decision scale/snapshot.

**`kubectl top` lỗi `Metrics API not available`:** metrics-server chưa cài
hoặc chưa sẵn sàng — kiểm tra `kubectl get apiservices | grep metrics` (trạng
thái phải `True`); trên minikube bật lại addon *(nhắc lại Bài 8 Phần
metrics-server cho HPA)*.

**Theo dõi rollout bằng watch:** `kubectl rollout status deploy/x` hoặc
`kubectl get pods -w -l app=x` — thấy pod mới Ready và pod cũ Terminating
theo thời gian thực, thay vì poll thủ công.

## References

- [Resource metrics pipeline](https://kubernetes.io/docs/concepts/cluster-administration/resource-metrics-pipeline/)
- [Tools for Monitoring Resources](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
