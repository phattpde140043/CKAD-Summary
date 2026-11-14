# Bài 3 — Kubernetes là gì

Tổng quan Kubernetes: bản chất là **cloud-native computing**, orchestration container
như thế nào, Kubernetes đến từ đâu, vì sao CNCF bảo đảm tính open source, kiến trúc
cluster và các API resource thiết yếu.

**Mục tiêu bài học:**

- Cloud-native computing là gì — vì sao đó là bản chất của Kubernetes
- Kubernetes cho phép cloud-native computing bằng cách nào (orchestration, scalability)
- Kubernetes đến từ đâu
- Kubernetes "thật sự" open source — được bảo đảm bởi Cloud Native Computing Foundation (CNCF)
- Kiến trúc Kubernetes & các API resource thiết yếu
- Lab cuối bài để thực hành

## Phần 1 — Cloud-native Computing

**Điểm xuất phát:** bản chất (essence) của Kubernetes là **cloud-native computing**.

**Cloud-native computing giải quyết vấn đề gì:**

- Mục tiêu: **cung cấp access tới ứng dụng trên internet** → ứng dụng cần được host trong cloud
- Khi host trong cloud, ứng dụng được **decoupled khỏi specific servers**: không muốn ứng dụng có quan hệ với bất kỳ server nào, vì như vậy là bị trói vào một **physical location** — không tốt
- Ứng dụng không gắn với server nào → cần **facilities** thay thế những gì từng do từng server riêng lo:

| Nhu cầu | Trên server riêng | Vấn đề khi lên cloud |
|---|---|---|
| **Configuration** | Config file nằm trên server | Tổ chức config thế nào khi ở cloud? |
| **Persistent storage** | Data lưu ngay trên server | App không nằm trên server thì lưu data ở đâu? |
| **Application access** | Truy cập thẳng server đang chạy app | Không address được server → cần cách access app trong cloud |

- **Cloud-native computing** là câu trả lời cho toàn bộ các vấn đề này

```
      ┌───────────────── Cloud ─────────────────┐
      │   Server 1     Server 2     Server 3    │
      └─────────────────────────────────────────┘
                        ▲
     Ứng dụng KHÔNG được quan hệ trực tiếp với server nào

   Máy local: đơn giản — fetch mọi thứ từ chính máy đó
   Cloud: cần giải pháp cho configuration · storage · access
          → Kubernetes lo bằng Kubernetes resources
          → resources THAY THẾ các files vốn nằm trên server riêng
```

- Đó chính là lý do cần tới các Kubernetes resources: chúng **thay thế các file** mà bình thường sẽ nằm trên server riêng của bạn

## Phần 2 — Kubernetes Orchestration

**Kubernetes là gì:**

- Open-source platform cho phép dùng **container trong môi trường cloud-native**
- Là **de facto standard** — theo tác giả: "không thực sự còn giải pháp nào khác trong open source"

**Orchestration:**

- Kubernetes **orchestrate container**: đảm bảo container **chạy đúng nơi cần chạy, với đúng số instance cần** — đó là tính **scalability**
- Scalability = đảm bảo **đủ số container đang chạy** để xử lý workload hiện tại
- Để hiện thực hóa, **Kubernetes API** định nghĩa một bộ **resource types** — pod, deployment, ConfigMap... — cho phép lưu thông tin trong môi trường cloud-native nơi không tồn tại quan hệ với server cụ thể *(nhìn trước — pod, deployment, ConfigMap học kỹ ở các module sau)*

**Orchestration hoạt động ra sao:**

- App trong cloud được Kubernetes **monitor** — luôn có gì đó đảm bảo app đang chạy
- Requests đổ về đột ngột tăng → orchestrate **availability**: app chạy **nhiều instance trên nhiều node**
- Có sự cố (một service/server biến mất) → orchestration đảm bảo app **chạy ở nơi khác**
- Kết quả: **không phải bận tâm** số lượng instance hay vị trí instance đang chạy nữa

## Phần 3 — Nguồn gốc Kubernetes

- **Google Borg** nằm ở nguồn gốc Kubernetes — platform Google dùng từ **đầu những năm 2000** để cung cấp Google applications trong môi trường cloud-based
- Dựa trên Borg, Google open-source một giải pháp **platform-agnostic** để orchestrate container
- Dòng thời gian: công bố lần đầu **tháng 6/2014** → donate source code cho **CNCF năm 2015**
- Vì sao thành công nhanh (theo tác giả): **reliability + uy tín của Google** — Borg vốn đang chạy các Google applications, donate cho open source nghĩa là cộng đồng nhận được một giải pháp **ready to use**

## Phần 4 — Kubernetes, CNCF & Distributions

**CNCF (Cloud Native Computing Foundation):**

- Là một phần của **Linux Foundation** — tổ chức nhiều kinh nghiệm hosting và nuôi dưỡng open source software
- Sứ mệnh: cung cấp **open source standards cho cloud-native computing**; hosts nhiều projects làm việc đó
- **Kubernetes là key project** — cung cấp platform chạy cloud-native applications

**Hệ sinh thái mở — và việc phải chọn:**

- CNCF tạo môi trường mở cho nhiều projects cùng đề xuất giải pháp cloud-native → hệ sinh thái lành mạnh, nhưng nhiều khi **nhiều project giải quyết cùng một vấn đề → phải chọn**
- Khi chạy Kubernetes: user chọn project/solution nào dùng **on top of Kubernetes** để cung cấp chức năng cụ thể
- **Vanilla Kubernetes** = core của hệ thống Kubernetes, **không tích hợp** project nào
- Hai đường đi:
  1. Tự chọn từng project và tích hợp vào vanilla Kubernetes
  2. Dùng **Kubernetes distribution** — nơi các project đã được tích hợp sẵn thành một môi trường Kubernetes hoạt động

**Demo cncf.io — thăm dò projects:**

| Quan sát | Ý nghĩa |
|---|---|
| Trang chủ [cncf.io](https://www.cncf.io) | Toàn quảng cáo **KubeCon** — Kubernetes conference chính của CNCF *(transcript chép "KubeConf" — tên đúng là KubeCon)* |
| Mục Projects, cột status | 4 trạng thái: **graduated** (mature hoàn toàn) · **incubating** (đang được giới thiệu) · **sandbox** (project mới) · **archived** (project cũ) |
| **etcd** | Core database của Kubernetes — tích hợp sẵn |
| **CoreDNS** | Lo DNS trong Kubernetes |
| **Helm** | Package manager |
| **CNI** | Network interface Kubernetes dùng — **incubating** (không nhất thiết phải graduated; incubating cũng đã khá mature) *(transcript chép "CNCI" — đúng là CNI)* |

**Đặc điểm của distribution:**

- **Opinionated**: mỗi chức năng chỉ tích hợp **một** project → user khỏi phải chọn. Lợi ích support: support engineer tập trung trọn vẹn vào một project → support tốt hơn
- Mở hơn: để user tự chọn giải pháp cụ thể
- Distribution có thể kèm **support** → chạy được Kubernetes ở nơi đòi hỏi reliability

**3 loại distribution:**

| Loại | Ví dụ |
|---|---|
| **Cloud-managed** (public cloud) | **EKS** — Amazon · **AKS** — Microsoft · **GKE** — Google |
| **On-premise** (data center / private cloud) | Google Anthos, Rancher, Red Hat OpenShift, Canonical Kubernetes |
| **Minimized** (testing & learning) | **Minikube** (dùng trong khóa này), K3s, OpenShift Local (**CRC** — môi trường học OpenShift thu nhỏ) |

- Phạm vi distribution không cố định vào một môi trường: OpenShift dùng được cả cloud lẫn on-premise
- **Minikube**: đến từ một Kubernetes **SIG** (special interest group); là learning environment nhưng theo tác giả là một **full Kubernetes distribution** — behave như distribution thật *(chi tiết lab environment ở Bài 4)*

## Phần 5 — Kiến trúc Kubernetes

**Control plane:**

- Nền tảng của kiến trúc — **một hoặc nhiều node** nơi chạy Kubernetes core services
- Lý tưởng là **nhiều node** để có redundancy khi một node down; production thường chạy nhiều server = **high availability** — rất phổ biến

| Core service | Vai trò |
|---|---|
| **kube-apiserver** | Cung cấp access tới API — component bạn "nói chuyện" khi dùng `kubectl` *(transcript chép "kubepi-server" — đúng là kube-apiserver)* |
| **etcd** | **Kubernetes database** — nơi lưu toàn bộ Kubernetes resources; là **central database** *(liên hệ Phần 4: etcd là CNCF project)* |
| **kube-scheduler** | **Scheduling pod** tới vị trí cụ thể — pickup resource và tìm **eligible nodes** |
| **kube-controller-manager** | Quản lý các **core Kubernetes processes** |

**Worker nodes:**

- Nơi chạy **containerized applications** (user applications), dùng 2 core services:

| Core service | Vai trò |
|---|---|
| **container runtime** | Phần **thực sự chạy container**; truy cập qua **CRI — Container Runtime Interface** |
| **kubelet** | Phần được **kube-scheduler liên hệ** để chạy các container trong pod |

**Có cần Docker/Podman trên các node không? — Không.**

- Docker/Podman là **container managers** chỉ hợp lý ở môi trường **standalone**; trong môi trường cloud/data center, **Kubernetes chính là container manager** → không cần Docker/Podman trên managed nodes

**Luồng hoạt động (drawing):**

```
              kubectl (user)
                    │
                    ▼
      ┌──────── CONTROL PLANE ─────────┐
      │ kube-apiserver                 │
      │ etcd (central database)        │
      │ kube-scheduler · controller-mgr│
      └───────────────┬────────────────┘
                      │ scheduler chọn eligible node
          ┌───────────┴────────────┐
          ▼                        ▼
   ┌─ WORKER NODE 1 ─┐      ┌─ WORKER NODE 2 ─┐
   │ kubelet + CRI   │      │ kubelet + CRI   │
   │  ┌─────────┐    │      │  ┌─────────┐    │
   │  │   Pod   │    │      │  │   Pod   │    │
   │  └─────────┘    │      │  └─────────┘    │
   └─────────────────┘      └─────────────────┘
```

1. Dùng `kubectl` → nói chuyện với **API server**
2. Tạo resource → resource được lưu vào **etcd**
3. **Scheduler** pickup resource, tìm các **eligible node**
4. Pod được schedule chạy trên worker node — một hoặc cả hai node, tùy mục đích

## Phần 6 — API Resources Thiết Yếu

- API resources là thành phần thiết yếu của Kubernetes: **định nghĩa cách ứng dụng hoạt động** — mọi thứ về storing & running applications trong môi trường K8s
- Liên hệ Phần 1: cloud-native không có quan hệ với server cụ thể → **không thể chỉ "đặt file đâu đó"** → API resources là cách thông minh để lưu thông tin
- Tác giả khuyên **memorize các resource types này càng sớm càng tốt** — đây là key terminology của cả khóa học

| Resource | Vai trò |
|---|---|
| **Pod** | **Entity tối thiểu** do Kubernetes quản lý; chạy **một hoặc nhiều container**. K8s start pod chứ **không start container** — "K8s không biết về container". **IP address** là tính năng quan trọng: IP thuộc về pod, **container không có IP** |
| **Deployment** | Cách **ưu tiên** để chạy ứng dụng — thêm **replication** + **zero-downtime application updates** cho pod |
| **ReplicaSet** | Xưa: dùng để quản lý replicas; **nay tự động được tạo** khi chạy ứng dụng dạng Deployment |
| **ConfigMap** | Lưu **configuration files + startup parameters** — "K8s equivalent của configuration file" |
| **Secret** | Closely related với ConfigMap — configuration dạng **Base64-encoded** |
| **Service** | **Load balance** incoming traffic tới các application instances |
| **Ingress** | **Reverse HTTP proxy** cho application access — nói chuyện với Service, đưa user tới ứng dụng; **ngày nay đang được thay bằng Gateway API** |
| **PersistentVolume (PV)** | **Persistent, non-ephemeral storage** — lưu files bên ngoài |

**Các resource phối hợp với nhau (drawing):**

```
                     Users
                       │
             Ingress (reverse HTTP proxy)
                       │
             Service (load balancer)
              ┌────────┴────────┐
            Pod 1            Pod 2   ← replicas — Deployment quản lý
              └────────┬────────┘       (tự tạo ReplicaSet bên dưới)
       ConfigMap · Secret · PV
       (config + storage cung cấp cho pod)
```

- Tất cả bắt đầu từ container — nhưng K8s không quản lý container: đặt container vào **pod** (entity nhỏ nhất K8s chạy được)
- Chạy nhiều instance của pod = **replicas** — quản lý bằng **Deployment**, resource để chạy **scalable application**
- Truy cập ứng dụng: **Service** load balance các instance; **Ingress** làm reverse proxy cho dễ access
- Đây chỉ là các resource thiết yếu — K8s còn **rất nhiều resource** khác để lưu mọi thứ cần lưu trong cloud *(học dần trong các module sau)*

> **Mẹo thi CKAD:** trên kỳ thi được phép truy cập tài liệu **[kubernetes.io/docs](https://kubernetes.io/docs/)** — cực kỳ hữu ích. Trong khóa, tác giả sẽ chỉ phần docs liên quan ở từng chủ đề và giải thích vì sao dùng được trên kỳ thi.

## Tổng hợp lệnh

```
minikube start                  # khởi động Kubernetes qua Minikube
kubectl api-resources           # liệt kê toàn bộ API resources
kubectl api-resources | less    # danh sách dài — pipe qua less để đọc
kubectl explain pod             # giải thích các field của resource pod
kubectl explain pod.spec        # sâu hơn: các parameter trong spec
```

- Tài liệu: [kubernetes.io/docs](https://kubernetes.io/docs/) — search tên resource (pod, service...) để xem description + YAML code *(được dùng trên kỳ thi)*
