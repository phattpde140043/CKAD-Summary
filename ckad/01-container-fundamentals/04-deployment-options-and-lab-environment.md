# Bài 4 — Kubernetes Deployment Options & Lab Environment

Dựng lab environment cho toàn bộ khóa học: tổng quan các lựa chọn triển khai
Kubernetes, cài Minikube trên Linux/Windows/macOS, chạy Kubernetes trên public
cloud, tự dựng all-in-one distribution bằng kubeadm, và chạy ứng dụng đầu tiên
bằng kubectl.

**Mục tiêu bài học:**

- Tổng quan các lựa chọn deploying Kubernetes
- Cài Minikube trên Linux, Windows hoặc macOS — và cách làm việc với nó
- Chạy Kubernetes trên public cloud
- Tự dựng all-in-one Kubernetes distribution
- Chạy ứng dụng đầu tiên
- Lab cuối bài để dựng lab environment riêng

> Khuyến nghị của tác giả: **dùng Minikube cho khóa này** — đó là distribution
> đầy đủ, gần giống môi trường thi nhất. Dùng lựa chọn khác có thể **thiếu
> chức năng** mà khóa học bao phủ.

## Phần 1 — Các Lựa Chọn Triển Khai Kubernetes

**Vì sao Minikube:**

- Học Kubernetes thì môi trường nào cũng dùng được — nhưng ôn CKAD cần môi trường **đầy đủ chức năng**, không mất thời gian tìm cách cấu hình → **Minikube là lựa chọn tốt nhất**
- Một số demo trong khóa chạy mượt trên Minikube nhưng **cần cấu hình thêm** trên distribution khác → tác giả ưu tiên mọi demo trên Minikube
- Trên kỳ thi, bạn làm việc với Kubernetes **từ hệ thống Ubuntu Linux** → cài Minikube trên **Ubuntu VM** để quen luôn Linux command line
- Lý thuyết thi không "đòi" biết Linux — nhưng thực tế phải **nắm chắc Linux cơ bản**; thiếu nó sẽ rất khó
- Ubuntu trong **WSL** (Windows Subsystem for Linux) cũng được hỗ trợ

**3 mô hình triển khai:**

| Mô hình | Control + worker node | Ai lo phần cài đặt |
|---|---|---|
| **On-premise** | Chạy trong data center của bạn — server/VM riêng | Tự cài |
| **Managed Kubernetes** (cloud) | Trên cloud platform | Cloud provider cài & setup toàn bộ |
| **AIO — all-in-one** (lớp học dùng) | Control + worker **cùng một VM**, thậm chí một container | Chỉ cần một lệnh start |

```
   ON-PREMISE                    MANAGED KUBERNETES (cloud)
   ┌─────────┐  ┌─────────┐      cloud provider cài đặt
   │ control │  │ worker  │      & vận hành sẵn — khỏi cài
   └─────────┘  └─────────┘
    data center / server / VM riêng
                     │
                     │  LỚP HỌC DÙNG: AIO (all-in-one)
                     ▼
        ┌──────────────────────────────┐
        │  1 VM (hoặc 1 container):    │
        │  control + worker + client   │  ← Minikube
        └──────────────────────────────┘
```

- Quan điểm "muốn học Kubernetes phải cài on-prem" là **sai**: cách install ứng dụng **giống hệt nhau** trên Minikube, on-prem hay cloud

## Phần 2 — Cài Đặt Minikube

> *(Transcript chép flag/lệnh theo kiểu đọc — `//cni is calico`, `–vmdriver is docker`, `kubedm`, `container-d`… — bài này đã chuẩn hóa về cú pháp đúng: `--cni=calico`, `--vm-driver=docker`, kubeadm, containerd.)*

**Cách 1 — Ubuntu VM (khuyến nghị):**

- Có thể **đã làm sẵn từ Lab 1.0** (script `minikube-docker-setup.sh` cài cả minikube lẫn docker)
- Làm manual: dựng VM **Ubuntu LTS Desktop mới nhất** bằng phần mềm ảo hóa ưa thích (chưa rành virtualization → xem khóa *Virtualization for Everyone* của tác giả), rồi:

| Bước | Lệnh |
|---|---|
| Cài git | `sudo apt install git` |
| Clone repo khóa học | `git clone https://github.com/sandervanvugt/ckad` |
| Cài Minikube + Docker | `./minikube-docker-setup.sh` (chạy từ thư mục ckad) |
| Khởi động | `minikube start --cni=calico` |
| Verify | `kubectl get all` — có output là OK |

- **Vì sao calico:** CNI là network interface; calico cung cấp chức năng mà CNI mặc định không có — tiêu biểu **network policy**, một chủ đề sẽ học trong khóa

**Cách 2 — Ubuntu trên Windows WSL:**

| Bước | Lệnh / việc cần làm |
|---|---|
| Cài **WSL version 2** | Bắt buộc v2 — v1 thiếu chức năng khóa này cần |
| Cài Docker Desktop cho Windows | [docs.docker.com/desktop/install/windows-install](https://docs.docker.com/desktop/install/windows-install/) |
| Kiểm tra | `wsl -l -v` — xác nhận WSL2 đang chạy Ubuntu |
| Chuyển sang v2 nếu chưa | PowerShell chạy admin: `wsl --set-version ubuntu-22.04` (hoặc 24.04 nếu có) |
| Cài đặt | Clone repo → chạy `minikube-docker-setup.sh` như Cách 1 |
| Khởi động | `minikube start --vm-driver=docker --cni=calico` |

**Cách 3 — Cài thẳng trên macOS / Windows: được, nhưng đừng:**

- Mất trọn trải nghiệm **Linux command line** — phần quan trọng cần quen trước khi thi
- Thật sự không còn cách nào khác: hướng dẫn cài tại [minikube.sigs.k8s.io/docs/start](https://minikube.sigs.k8s.io/docs/start/) *(URL hiện trên màn hình video nhưng không có trong transcript — bổ sung từ tài liệu chính thức của Minikube)*

## Phần 3 — Kubernetes Trên Public Cloud

- Kubernetes chạy được trên **mọi public cloud platform**; cloud provider thường lo sẵn các chức năng cụ thể: xử lý incoming traffic qua **Ingress**, giới hạn traffic bằng **network policies**, **storage allocation**
- Đẹp — nhưng vì các chức năng bị cloud "ôm" sẵn nên **kém phù hợp để học** → không khuyến nghị cho ôn CKAD
- Nếu vẫn muốn học trên public cloud: deploy **Ubuntu VM trên cloud rồi cài Minikube bên trong** (như Phần 2); public cloud networking phức tạp → vẫn không khuyến nghị
- Tác giả cố tình không demo giải pháp này

## Phần 4 — All-in-one bằng kubeadm (tùy chọn, nâng cao)

- Dành cho người **đã có kinh nghiệm Kubernetes** và muốn thử thách hơn: cấu hình Ubuntu LTS server VM làm **control node + worker node + client cùng lúc**
- Cách làm giống hệt dựng **cluster thật** — dùng utility **kubeadm**
- ⚠️ **Không nằm trong yêu cầu thi CKAD** — chỉ để luyện thêm

**Các bước demo** (SSH từ Ubuntu desktop vào Ubuntu server):

| Bước | Lệnh | Kết quả / Verify |
|---|---|---|
| 1 | `git clone https://github.com/sandervanvugt/ckad` | Lấy repo khóa học. Trong repo có nhiều shell script — chỉ dùng `setup-container.sh` + `setup-kubetools.sh`, bỏ qua các script khác |
| 2 | `sudo ./setup-container.sh` | Setup **containerd** (phần "Linux" nhiều hơn "Kubernetes" — nên tác giả viết sẵn script). **Verify:** `systemctl status containerd` → active (running) |
| 3 | `sudo ./setup-kubetools.sh` | Cài **kubectl, kubeadm, kubelet** trực tiếp từ Kubernetes repositories |
| 4 | `sudo kubeadm init` | Tạo cluster — **5+ phút** tùy mạng. Khi xong output chỉ ra 3 việc: setup client · deploy pod network · join worker node (AIO chỉ cần 2 việc đầu) |
| 5 | Paste 3 lệnh setup client in ra từ `kubeadm init` (mkdir + copy `admin.conf` về `~/.kube/config` + chown) | **Verify:** `kubectl get all` trả về thông tin cluster |
| 6 | `kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml` | Cài **pod network** — không có nó cluster không chạy được pod |
| 7 | `kubectl get pods -n kube-system` | Theo dõi core system pods; cần **calico-node** + **CoreDNS** Running (lúc đầu ContainerCreating, chờ thêm) |
| 8 | `kubectl run testpod --image=nginx` → `kubectl get pods` | **Pending** — control node bình thường **không chạy user pod** |
| 9 | `kubectl edit node kubeaio2` *(tên node trong demo)* | Tìm **taint** — restriction chặn node chạy pod — **xóa 3 dòng taint**, thoát editor |
| 10 | `kubectl get pods` | testpod: Pending → ContainerCreating (đang pull image) → **Running** |

- Cảnh báo của tác giả: dùng môi trường này trong khóa = phải **tự cài** ingress controller, storage provisioner… *(học ở module sau)* — không dễ như Minikube: "for experienced people only"

## Phần 5 — Ứng Dụng Đầu Tiên & kubectl

**Web dashboard — dùng được nhưng bỏ qua:**

- Nhiều distribution cung cấp web-based Kubernetes dashboard; vui nhưng **không dùng được trên kỳ thi** (có thể được cài, nhưng chức năng hạn chế)
- Không thể pass CKAD chỉ bằng dashboard → **phải thành thạo command line**

**kubectl:**

- **Core utility** cho Kubernetes — utility quan trọng nhất, gần như duy nhất cho CKAD; ngoài ra cần một chút về **helm** *(học ở module sau)*
- Bật tab completion ngay từ đầu (bash; môi trường khác xem docs):

```
source <(kubectl completion bash)
```

- `-h` cho help **ở mọi cấp** — kubectl **không có man pages**, `-h` thay thế tất cả
- **Examples nằm ở đầu** output `-h` — tác giả khuyên dùng nhiều vì rất tiện

**Demo — tạo deployment đầu tiên:**

| Lệnh | Kết quả |
|---|---|
| `kubectl -h \| less` | Toàn bộ lệnh của kubectl — pipe qua less vì rất dài |
| `kubectl create -h \| less` | Examples ở đầu + các subcommand của `kubectl create` |
| `kubectl create deployment -h \| less` | Examples + mọi options (viết gọn `deploy` cũng được) |
| `kubectl completion -h` | Chính nó chứa lệnh `source <(kubectl completion bash)` — khỏi cần nhớ thuộc lòng |
| `source <(kubectl completion bash)` | Tab 2 lần sau `kubectl create ` → hiện mọi subcommand; `kubectl create deployment ` → gợi ý `--image=` + tên |
| `kubectl create deployment firstapp --image=nginx` | `deployment.apps/firstapp created` |
| `kubectl get all` | **Verify:** thấy cả 3 — deployment + **replicaset** (do deployment quản lý, lo scalability) + **1 pod instance** |

- `deploy` = deployment — **layer thêm vào pod** để có scalability + update strategy *(liên hệ Bài 3 Phần 6)*

> **Mẹo thi CKAD:** mọi thao tác phải qua command line (dashboard không dùng được trên thi). Tab completion + `kubectl -h` với examples ở đầu output là hai "vũ khí" tiết kiệm thời gian — tác giả khuyên dùng completion "all the time".

## Tổng hợp lệnh

```
# Minikube (lab environment)
minikube start --cni=calico     # khởi động Minikube với CNI calico (có network policy)
minikube start -h               # rất nhiều option: --cni, --cpus, --memory, --disk-size
kubectl get all                 # kiểm tra tổng thể: có output = environment OK

# kubectl
source <(kubectl completion bash)                  # bật tab completion (bash)
kubectl -h                                         # help ở mọi cấp — không có man pages
kubectl create deployment firstapp --image=nginx   # tạo deployment đầu tiên
kubectl get all                                    # deployment + replicaset + pod

# kubeadm AIO (tùy chọn — không thuộc đề CKAD)
sudo kubeadm init                # tạo cluster (5+ phút)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml   # cài pod network
kubectl get pods -n kube-system  # calico-node + CoreDNS phải Running
kubectl edit node <node>         # xóa taint để control node chạy được user pod
```
