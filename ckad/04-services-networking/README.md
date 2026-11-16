# Module 4 — Services and Networking

Module về **exposing applications**: kết nối traffic từ bên ngoài vào ứng dụng
đang chạy trong cluster — Services, Ingress, Gateway API. Miền **Services and
Networking** chiếm 20% đề thi. Tóm tắt từ video course CKAD của Sander van
Vugt — tương ứng course *Certified Kubernetes CKAD: Services and Networking*.

## Bài học

Mỗi bài học một file riêng; bài chưa có link nghĩa là chưa tạo file. Danh sách
bài được cập nhật dần khi xử lý transcript từng bài.

- [Bài 10 — Networking](10-networking.md): cluster network/pod network,
  ClusterIP & NodePort, `kubectl expose`, DNS & FQDN, NetworkPolicy, Gateway
  API & Istio overview + end-of-lesson lab
- [Bài 11 — Ingress and Gateway API](11-ingress-and-gateway-api.md): Ingress
  controller (Helm + minikube addon), Ingress rules & pathType, Gateway API
  (GatewayClass/Gateway/HTTPRoute), troubleshooting networking +
  end-of-lesson lab

## Labs

- [Labs thực hành](labs.md) — tổng hợp toàn bộ ví dụ demo và lab exercise của module, dạng checklist kèm tiêu chí verify.

## Deep dive

Bổ sung kiến thức ngoài video theo template 4 lớp (Concept → YAML →
Command → Production), bám checklist CNCF của domain:

- [dd-ingress-advanced.md](dd-ingress-advanced.md) — ingressClassName,
  defaultBackend, TLS termination

## Tiến độ

- [x] Bài 10: Networking
- [x] Bài 11: Ingress and Gateway API
