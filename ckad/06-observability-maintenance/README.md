# Module 6 — Observability & Maintenance

Module về **quan sát và troubleshoot workload**: chẩn đoán sự cố theo
phương pháp tổng quát, phân tích ứng dụng fail, sự cố network,
authentication issues, debugging bằng Linux skills — cùng observability.
Miền **Application Observability and Maintenance** chiếm 15% đề thi. Tóm
tắt từ video course CKAD của Sander van Vugt — tương ứng Module 6 của
course *Certified Kubernetes Application Developer (CKAD), 4th Edition*.

## Bài học

Mỗi bài học một file riêng; bài chưa có link nghĩa là chưa tạo file. Danh sách
bài được cập nhật dần khi xử lý transcript từng bài.

- [Bài 16 — Troubleshooting](16-troubleshooting.md): phương pháp
  troubleshooting tổng quát, phân tích ứng dụng fail, sự cố network
  access, cluster event logs, authentication issues, debugging bằng
  Linux skills + end-of-lesson lab
- [Bài 17 — Observability](17-observability.md): observability —
  logs/metrics/traces, API health endpoints, probes (liveness /
  readiness / startup) + end-of-lesson lab

## Labs

- [Labs thực hành](labs.md) — tổng hợp toàn bộ ví dụ demo và lab exercise của module, dạng checklist kèm tiêu chí verify.

## Deep dive

Bổ sung kiến thức ngoài video theo template 4 lớp (Concept → YAML →
Command → Production), bám checklist CNCF của domain:

- [dd-logs.md](dd-logs.md) — đầy đủ flags `kubectl logs` + đọc log crash
- [dd-metrics-events-watch.md](dd-metrics-events-watch.md) — `kubectl
  top`, events, watch
- [dd-grpc-probes.md](dd-grpc-probes.md) — probe handler thứ tư cho app
  gRPC
- [dd-kubectl-debug.md](dd-kubectl-debug.md) — ephemeral containers:
  debug distroless và node

## Tiến độ

- [x] Bài 16: Troubleshooting
- [x] Bài 17: Observability
