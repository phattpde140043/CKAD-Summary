# Quiz — Module 1: Container Fundamentals

Trắc nghiệm tự test 2 bài lý thuyết chính của module *(bổ sung từ nghiên cứu
ngoài video — câu hỏi tổng hợp thêm, nội dung đối chiếu trực tiếp với notes)*.

**Cách dùng:** làm hết 10 câu **không nhìn notes** → đối chiếu phần
*Đáp án & giải thích* ở cuối file → câu sai thì mở Phần bài học tương ứng đọc
lại. Mục tiêu: 10/10 trước khi sang Module 2.

## Câu hỏi — Bài 3: Kubernetes là gì

**Câu 1.** Vai trò chính của CNCF (Cloud Native Computing Foundation) đối với Kubernetes là gì?

- A. Phát triển các phần mềm cloud-native độc quyền (proprietary)
- B. Cung cấp open source standards cho cloud-native computing; quản lý & bảo trợ các projects liên quan
- C. Cung cấp dịch vụ hỗ trợ thương mại cho các bản phân phối Kubernetes
- D. Xây dựng hạ tầng cloud để lưu trữ và vận hành ứng dụng

**Câu 2.** Vai trò chính của kube-apiserver trong kiến trúc Kubernetes là gì?

- A. Lưu trữ tất cả Kubernetes resources
- B. Chạy các ứng dụng được đóng gói trong container
- C. Cung cấp giao diện tương tác với Kubernetes qua các công cụ như `kubectl`
- D. Quản lý việc lập lịch pod trên worker node

**Câu 3.** Đâu **không** phải thách thức mà cloud-native computing hướng tới giải quyết?

- A. Quản lý configuration cho ứng dụng
- B. Persistent storage cho ứng dụng
- C. Truy cập ứng dụng không phụ thuộc vào server cụ thể nào
- D. Tăng tốc độ xử lý của ổ đĩa cục bộ (local disk)

**Câu 4.** Một Kubernetes distribution mang tính "opinionated" là gì?

- A. Distribution tích hợp nhiều CNCF projects và để user tự chọn project muốn dùng
- B. Distribution chỉ tập trung vào môi trường cloud được quản lý
- C. Distribution tích hợp sẵn một tập hợp projects cụ thể, hạn chế lựa chọn của user để đảm bảo hỗ trợ & tương thích tốt hơn
- D. Distribution không hỗ trợ bất kỳ CNCF project nào

**Câu 5.** Mục tiêu chính của cloud-native computing trong bối cảnh triển khai ứng dụng là gì?

- A. Gắn ứng dụng với các máy chủ vật lý cụ thể để cải thiện hiệu năng
- B. Tách ứng dụng khỏi sự phụ thuộc vào máy chủ cụ thể; truy cập & sử dụng tài nguyên linh hoạt
- C. Giảm sử dụng tài nguyên cloud bằng cách hạn chế triển khai ứng dụng
- D. Tăng bảo mật bằng cách chỉ lưu trữ ứng dụng trên máy chủ cục bộ

## Câu hỏi — Bài 4: Deployment Options & Lab Environment

**Câu 6.** Lý do chính Minikube được khuyến nghị để học Kubernetes, đặc biệt khi ôn CKAD?

- A. Là môi trường duy nhất hỗ trợ Kubernetes
- B. Dễ cấu hình và hoạt động ổn định với hầu hết các bài thực hành trong khóa học
- C. Cung cấp hiệu năng tốt hơn các môi trường Kubernetes trên cloud
- D. Yêu cầu kiến thức Linux chuyên sâu để cài đặt

**Câu 7.** Khác biệt chính giữa managed Kubernetes và giải pháp AIO (all-in-one) như Minikube là gì?

- A. Managed Kubernetes chỉ có trên on-premise, AIO chạy trên cloud
- B. Managed Kubernetes do cloud provider thiết lập & bảo trì; AIO chạy cả control plane lẫn worker node trong một VM/container duy nhất
- C. AIO đòi hỏi kiến thức cài đặt nâng cao hơn managed Kubernetes
- D. Managed Kubernetes chỉ dùng để kiểm thử, AIO dùng cho production

**Câu 8.** Vì sao nên dựng Minikube trên Ubuntu VM thay vì cài thẳng trên macOS/Windows?

- A. Minikube không hoạt động trên macOS hay Windows
- B. Ubuntu có hiệu năng tốt hơn macOS/Windows
- C. Trải nghiệm Linux command line rất quan trọng đối với kỳ thi CKAD
- D. Ubuntu là hệ điều hành duy nhất Minikube hỗ trợ

**Câu 9.** Mục đích của option `--cni=calico` khi chạy `minikube start` là gì?

- A. Chỉ định Container Runtime Interface mà Minikube dùng để quản lý vòng đời pod
- B. Cấu hình Minikube dùng Calico làm giải pháp networking, hỗ trợ Network Policies
- C. Bật giao diện đồ họa để quản lý tài nguyên cluster
- D. Thiết lập VM driver mặc định và cấu hình network stack cho Minikube

**Câu 10.** Mục đích chính của lệnh `kubeadm init` trong quá trình thiết lập Kubernetes là gì?

- A. Cài đặt các công cụ CLI của Kubernetes như `kubectl`
- B. Khởi tạo và thiết lập một Kubernetes cluster
- C. Triển khai network plugin cho cluster
- D. Khởi động container runtime trên các node

## Đáp án & giải thích

### Bài 3 — đáp án: B · C · D · C · B

| Câu | Đáp án | Kiến thức then chốt | Đọc lại |
|---|---|---|---|
| 1 | **B** | CNCF thuộc **Linux Foundation**; sứ mệnh: open source standards + hosting các cloud-native projects (Kubernetes, Prometheus, Envoy, Helm...). Không phải nhà cung cấp thương mại | [Bài 3](03-kubernetes-overview.md) Phần 4 |
| 2 | **C** | kube-apiserver là **gateway** mọi request đi qua: kubectl → authentication → authorization → etcd. Không chạy container (kubelet/CRI lo) và không schedule pod (kube-scheduler lo) | [Bài 3](03-kubernetes-overview.md) Phần 5 |
| 3 | **D** | Thách thức cloud-native: configuration → ConfigMap/Secret · persistent storage → PV/PVC · truy cập không phụ thuộc server → Service/Ingress · autoscaling → HPA. Tối ưu local disk là bài toán hạ tầng, không phải cloud-native | [Bài 3](03-kubernetes-overview.md) Phần 1 & 6 |
| 4 | **C** | Opinionated: mỗi chức năng tích hợp **một** project sẵn → user khỏi phải chọn, support tập trung (OpenShift là ví dụ điển hình). Ngược với vanilla/upstream — linh hoạt hơn nhưng phải tự quyết định mọi thành phần | [Bài 3](03-kubernetes-overview.md) Phần 4 |
| 5 | **B** | Ứng dụng **decoupled khỏi server cụ thể**: Service giữ endpoint ổn định, pod được reschedule khi node down. Lưu ý: muốn tận dụng được cần đủ replicas + readiness probe *(học ở module sau)* | [Bài 3](03-kubernetes-overview.md) Phần 1–2 |

### Bài 4 — đáp án: B · B · C · B · B

| Câu | Đáp án | Kiến thức then chốt | Đọc lại |
|---|---|---|---|
| 6 | **B** | Minikube là **full distribution** đầy đủ chức năng, gần môi trường thi nhất; mọi demo của khóa chạy mượt mà không cần cấu hình thêm — các distribution khác có thể thiếu chức năng | [Bài 4](04-deployment-options-and-lab-environment.md) Phần 1 |
| 7 | **B** | Managed: cloud provider cài & vận hành (EKS/AKS/GKE). AIO: control + worker **cùng một VM/container**, khởi động bằng một lệnh. Nhưng cách install ứng dụng thì **giống hệt nhau** ở mọi mô hình | [Bài 4](04-deployment-options-and-lab-environment.md) Phần 1 |
| 8 | **C** | Kỳ thi CKAD làm việc với Kubernetes **từ hệ thống Ubuntu Linux** → quen Linux command line là lợi thế trực tiếp. Minikube vẫn hỗ trợ macOS/Windows nên A, D sai | [Bài 4](04-deployment-options-and-lab-environment.md) Phần 1–2 |
| 9 | **B** | CNI mặc định **thiếu network policy** — Calico bổ sung đúng phần này (chủ đề sẽ học trong khóa). Phân biệt: **CRI** = interface tới container runtime (containerd) · **CNI** = interface networking · **Calico** = một CNI plugin cụ thể | [Bài 4](04-deployment-options-and-lab-environment.md) Phần 2 |
| 10 | **B** | `kubeadm init` khởi tạo control plane + cluster (5+ phút). **Không** cài kubectl/kubelet/runtime/CNI — demo dùng `setup-container.sh`, `setup-kubetools.sh` và `calico.yaml` cho các việc đó. Nhớ: kubeadm không thuộc đề thi CKAD | [Bài 4](04-deployment-options-and-lab-environment.md) Phần 4 |

## Từ khóa phải thuộc

| Thuật ngữ | Ghi nhớ nhanh |
|---|---|
| Cloud-native computing | Ứng dụng decoupled khỏi server cụ thể; config · storage · access do K8s resources lo |
| CNCF | Tổ chức thuộc Linux Foundation — standards + hosting cloud-native projects |
| kube-apiserver | Gateway mọi request (kubectl) — auth → authz → etcd |
| etcd | Central database của cluster |
| kube-scheduler | Chọn eligible node để schedule pod |
| kubelet | Agent trên mỗi node — chạy container trong pod (qua CRI) |
| Opinionated distribution | Tích hợp sẵn 1 project cho mỗi chức năng → khỏi chọn, support tốt |
| Minikube | AIO distribution — control + worker trong 1 VM/container; full distribution gần môi trường thi nhất |
| Managed Kubernetes | EKS/AKS/GKE — cloud provider cài đặt & vận hành |
| CRI | Container Runtime Interface — tới container runtime (containerd, CRI-O) |
| CNI | Container Network Interface — networking cho pod (Calico, Cilium, Flannel); Calico hỗ trợ NetworkPolicy |
| kubeadm | Bootstrap cluster thật (`init` control plane, `join` worker) — ngoài đề CKAD |

> **Phân định phạm vi khi ôn:** kubeadm, CNI, control plane ops thiên về **CKA**; CKAD tập trung **chạy ứng dụng** — pod, deployment, service, ConfigMap/Secret, probes. Đừng sa đà vào nội dung ngoài đề.
