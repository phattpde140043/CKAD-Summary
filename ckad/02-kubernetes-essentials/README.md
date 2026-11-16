# Module 2 — Kubernetes Essentials

Module đầu tiên dạy các khái niệm **trực tiếp liên quan kỳ thi CKAD**. Ứng dụng
Kubernetes luôn chạy dưới dạng **Pod** — hai bài đầu tiên của module tập trung
quản lý toàn bộ các tính năng liên quan của Pod *(transcript ghi "learning as
pods" — lỗi transcribe của "running as pods")*; bài tiếp theo dạy cách dùng
**Storage** trong môi trường cloud-native. Tóm tắt từ video course CKAD của
Sander van Vugt — tương ứng course *Certified Kubernetes CKAD: Kubernetes
Essentials*.

## Bài học

Mỗi bài học một file riêng; bài chưa có link nghĩa là chưa tạo file. Danh sách
bài được cập nhật dần khi xử lý transcript từng bài.

- [Bài 5 — Understanding and Using Pods](05-understanding-and-using-pods.md): lựa chọn chạy ứng dụng (Pod · Deployment · StatefulSet · Helm), Pod là gì, YAML declarative/imperative, generate YAML, multi-container pod, namespaces, troubleshooting + end-of-lesson lab
- [Bài 6 — Advanced Pod Features](06-advanced-pod-features.md): init containers, sidecar container (K8s ≥ 1.29), port forwarding, restart policy, jobs & cron jobs, clean up resources + end-of-lesson lab
- [Bài 7 — Storage](07-storage.md): container storage ephemeral, pod volumes (emptyDir, hostPath), persistent volumes (PV), persistent volume claims (PVC), storage class, cấu hình pod storage bằng PVC + end-of-lesson lab

## Labs

- [Labs thực hành](labs.md) — tổng hợp toàn bộ ví dụ demo và lab exercise của module, dạng checklist kèm tiêu chí verify.

## Deep dive

Bổ sung kiến thức ngoài video theo template 4 lớp (Concept → YAML →
Command → Production), bám checklist CNCF của domain:

- [dd-jobs-cronjobs-advanced.md](dd-jobs-cronjobs-advanced.md) —
  backoffLimit, activeDeadlineSeconds, suspend, timeZone, 3 concurrency
  policies

## Tiến độ

- [x] Bài 5: Understanding and Using Pods
- [x] Bài 6: Advanced Pod Features
- [x] Bài 7: Storage
