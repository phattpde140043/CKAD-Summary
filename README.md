# CKAD Summary — Certified Kubernetes Application Developer

Notes ôn tập cho kỳ thi Certified Kubernetes Application Developer (CKAD).

## Giới thiệu khóa học (Sander van Vugt)

Phần tóm tắt dưới đây lấy từ video course CKAD của Sander van Vugt.

**Course materials:** [github.com/sandervanvugt/ckad](https://github.com/sandervanvugt/ckad.git) —
repo chính thức của khóa, chứa lab files, Dockerfile demo, và script cài môi trường
(`minikube-docker-setup.sh`); clone một lần ở đầu khóa (xem Lab 1.0).

**Về chứng chỉ CKAD:**

- CKAD (Certified Kubernetes Application Developer) chứng minh kỹ năng **chạy ứng dụng trên Kubernetes cluster có sẵn** — không cần tự xây dựng cluster
- Tác giả nhận định CKAD thực chất nên gọi là "Application Deployer": tập trung vào chạy pod, deployment, đảm bảo ứng dụng **scalable** và **available**
- Phân biệt các chứng chỉ: **KCNA** (entry-level, trắc nghiệm) → **CKAD** và **CKA** (chứng chỉ cốt lõi, thực hành)

**Cấu trúc khóa học (7 module):**

| Module | Nội dung |
|--------|----------|
| 1 | [Container Fundamentals](ckad/01-container-fundamentals/): Containers, Container Images, Kubernetes là gì, Deployment Options & Lab |
| 2 | [Kubernetes Essentials](ckad/02-kubernetes-essentials/): lab environment, Pod, Storage |
| 3 | [Application Deployment](ckad/03-application-deployment/): Deployments, scalability, updates, StatefulSet, DaemonSet, cài đặt ứng dụng |
| 4 | Services & Networking: Ingress, Gateway API |
| 5 | Cấu hình môi trường & Security: ConfigMaps, Secrets, làm việc với API |
| 6 | Observability & Maintenance: troubleshooting |
| 7 | Sample exam — kèm **script chấm điểm pass/fail** để tự đánh giá trước khi thi thật |

**Điểm nổi bật:** Mỗi bài học đều có demo + lab exercise kèm lời giải.

**Về tác giả:** Sander van Vugt (Hà Lan) — chuyên gia Linux/Kubernetes/Ansible,
tác giả nhiều best-seller như *Getting Started with Kubernetes*, *Linux Under
the Hood*, đồng thời founder của Living Open Source Foundation (tổ chức phi lợi
nhuận dạy open source miễn phí ở châu Phi).

## Tổng quan kỳ thi

| Hạng mục | Chi tiết |
|---|---|
| Thời lượng | 2 giờ |
| Hình thức | Online, giám sát trực tuyến, thực hành trực tiếp trên command line |
| Phiên bản Kubernetes | v1.37 |
| Số lần thi | 2 lần thi + 2 phiên killer.sh simulator (17 câu mỗi phiên) |
| Hiệu lực | 2 năm |
| Điều kiện tiên quyết | Không |
| Chi phí | $445 (chỉ kỳ thi) |

Nguồn: [CNCF curriculum](https://github.com/cncf/curriculum/tree/master/ckad) ·
[Linux Foundation exam page](https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/) — xác nhận 2026-11-10.

## Miền kiến thức & trọng số

| # | Chủ đề | Trọng số |
|---|---|---|
| 1 | [Application Design and Build](01-application-design-and-build.md) | 20% |
| 2 | [Application Deployment](02-application-deployment.md) | 20% |
| 3 | [Application Observability and Maintenance](03-application-observability-and-maintenance.md) | 15% |
| 4 | [Application Environment, Configuration and Security](04-application-environment-configuration-security.md) | 25% |
| 5 | [Services and Networking](05-services-and-networking.md) | 20% |

## Cách sử dụng

Mỗi file chủ đề có một section cho từng competency chính thức, kèm checklist
khái niệm then chốt. Thêm notes trực tiếp dưới mục checklist tương ứng; đánh dấu
khi đã nắm vững.

Toàn bộ notes được biên soạn theo [WRITING_GUIDELINES.md](WRITING_GUIDELINES.md) —
quy ước ngôn ngữ, cấu trúc bài học, marker nguồn và format labs.
