# Module 5 — Application Environment, Configuration and Security

Module về **cấu hình môi trường ứng dụng và security**: đưa configuration
và dữ liệu nhạy cảm vào ứng dụng bằng ConfigMaps và Secrets, triển khai ứng
dụng theo cách DevOps, làm việc trực tiếp với Kubernetes API và các tùy chọn
security. Miền **Application Environment, Configuration and Security** chiếm
25% đề thi. Tóm tắt từ video course CKAD của Sander van Vugt — tương ứng
Module 5 của course *Certified Kubernetes Application Developer (CKAD),
4th Edition*.

## Bài học

Mỗi bài học một file riêng; bài chưa có link nghĩa là chưa tạo file. Danh sách
bài được cập nhật dần khi xử lý transcript từng bài.

- [Bài 12 — ConfigMaps and Secrets](12-configmaps-and-secrets.md): decoupling
  cấu hình khỏi application code — variables & `kubectl set env`, ConfigMap
  cho biến và configuration file, Secrets, Docker registry access secrets +
  end-of-lesson lab
- [Bài 13 — Deploying Applications the DevOps Way](13-deploying-applications-the-devops-way.md):
  DevOps & GitOps, blue-green và canary deployments qua Service và Ingress —
  zero-downtime upgrades + end-of-lesson lab
- [Bài 14 — Working With the API](14-working-with-the-api.md): Kubernetes
  API vận hành ra sao — api-resources/api-versions, `kubectl config`, curl
  qua `kubectl proxy`, API deprecations, mở rộng API bằng CRD và operators +
  end-of-lesson lab
- [Bài 15 — Security](15-security.md): authentication & authorization,
  service accounts, role-based access control, security context, resource
  requests/limits và quota + end-of-lesson lab

## Labs

- [Labs thực hành](labs.md) — tổng hợp toàn bộ ví dụ demo và lab exercise của module, dạng checklist kèm tiêu chí verify.

## Deep dive

Bổ sung kiến thức ngoài video theo template 4 lớp (Concept → YAML →
Command → Production), bám checklist CNCF của domain:

- [dd-resource-quota-limitrange.md](dd-resource-quota-limitrange.md) —
  ResourceQuota + LimitRange: 3 lớp phòng thủ cho namespace
- [dd-configmap-secret-advanced.md](dd-configmap-secret-advanced.md) —
  immutable, `type: tls`, `stringData`
- [dd-qos-classes.md](dd-qos-classes.md) — Guaranteed/Burstable/
  BestEffort, OOMKilled vs Evicted

## Tiến độ

- [x] Bài 12: ConfigMaps and Secrets
- [x] Bài 13: Deploying Applications the DevOps Way
- [x] Bài 14: Working With the API
- [x] Bài 15: Security
