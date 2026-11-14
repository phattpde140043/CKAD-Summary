# Bài 1 — Understanding and Using Containers

Tổng quan nhanh các kiến thức cốt lõi về container (*container essentials*) — không phải introduction đầy đủ, mà
là những gì cần biết khi làm việc với container trong môi trường Kubernetes.

**Mục tiêu bài học:**

- Container là gì và khác thế nào so với virtual machine
- Vai trò của container registry
- Cách khởi tạo và quản lý container
- Sử dụng container logs
- Lab cuối bài để thực hành các kỹ năng trên

## Phần 1 — Container là gì

**Định nghĩa:**

- Container cung cấp một cách chạy ứng dụng hoàn toàn cô lập (**isolation**). systemd vốn cũng chạy được ứng dụng, nhưng điểm mới của container là isolation toàn phần
- Để cung cấp khả năng cô lập này, Linux sử dụng tính năng Namespaces của kernel. Namespace cho phép một ứng dụng chạy với **góc nhìn riêng (view)** về tài nguyên hệ thống — không nhìn thấy tiến trình/tài nguyên của ứng dụng khác. Namespaces **không phải cơ chế cô lập duy nhất** — isolation đầy đủ còn cần **cgroups** (giới hạn CPU/RAM), **capabilities** (quyền) *(bổ sung từ nghiên cứu ngoài video — xem bảng runtime dưới đây)*
- Container đóng gói **toàn bộ dependencies** của ứng dụng bên trong namespace
- Container khởi chạy từ **image**, image thường lấy từ **registry**. Image được chuẩn hóa cao → dùng giống nhau trên máy standalone lẫn cloud (lý do học container trước Kubernetes)

**Các lớp kiến trúc:**

| Thành phần | Vai trò |
|---|---|
| **Container runtime** | Chương trình khởi chạy container (analog: systemd khởi chạy app → runtime khởi chạy container) |
| **Container engine** (thuật ngữ chính thức; tác giả gọi là *container manager*) | Quản lý container, nằm **trên** runtime — standalone: **Docker**, **podman** |
| **Container orchestrator** | Danh mục cao hơn engine: quản lý container trên toàn cluster — cloud: **Kubernetes** *(bổ sung từ nghiên cứu ngoài video: tách khỏi hàng engine để tránh hiểu Kubernetes là container engine như Docker/podman)* |
| **runc** | Phần lõi trung tâm — nhiệm vụ duy nhất: chạy container image. Cùng một runc + cùng một image chạy được cả ở máy local lẫn cloud |

- Kubernetes làm nhiều hơn việc chạy container → còn được gọi là **container orchestrator**
- Sức mạnh của container: **cùng 1 image** chạy với Docker/podman trên máy cá nhân hoặc với Kubernetes trong cloud

```
   Container Image (từ registry)
              │
            runc            ← runtime: chạy image thành container
              │
   ┌──────────┴──────────┐
 Docker / Podman      Kubernetes   ← quản lý container (engine / orchestrator)
 (standalone)         (cloud — orchestrator)
```

**Phân biệt các thuật ngữ runtime** *(bổ sung — làm rõ từ bị chép nhầm thành "OpenC" trong transcript; kiểm tra lại video: từ đúng là **runc**)*:

| Thuật ngữ | Bản chất |
|---|---|
| **OCI** (Open Container Initiative) | *Không phải runtime* — tổ chức tiêu chuẩn mở gồm: Image Spec (cấu trúc image), Runtime Spec (cách chạy container), Distribution Spec (cách phân phối image) |
| **runc** | Low-level runtime tuân thủ OCI — trực tiếp tạo/chạy container bằng Linux **namespaces** (cô lập), **cgroups** (giới hạn CPU/RAM), **capabilities** (quyền) |
| **containerd** | Runtime phổ biến nhất (nằm dưới Docker); Kubernetes thường dùng containerd + runc |
| **CRI-O** | Runtime thay thế containerd, thiết kế riêng cho Kubernetes, tuân thủ CRI (Container Runtime Interface) |

Vị trí runc trong kiến trúc Kubernetes *(nhìn trước — chi tiết ở Module 2)*:

`kubelet → (CRI) → containerd → runc → Linux kernel (namespaces + cgroups)`

> Lưu ý: containerd + runc là kiến trúc phổ biến nhất nhưng không phải cluster Kubernetes nào cũng dùng nó.

## Phần 2 — Container Registry và Docker Hub

**Container registry** = nơi lưu trữ và phân phối container image:

| Loại | Đặc điểm |
|---|---|
| **Local registry** | Private — chạy trên máy cá nhân hoặc network nội bộ |
| **Public registry** | Phục vụ người dùng toàn cầu. Phổ biến nhất: **Docker Hub** (thường là default registry) — chứa hàng triệu image |

**FQCN (Fully Qualified Container Name):**

- Tên đầy đủ của **container image**, bao gồm cả registry lấy image — dùng được trong mọi trường hợp khi cần chỉ định registry cụ thể
- Lưu ý: đây là tên *image* (image + registry), không phải tên *container*

Ví dụ: `eaeraregistry.azurecr.io/backend/crm-api:v2.1` — gồm 3 thành phần:

| Thành phần | Ví dụ | Ý nghĩa |
|---|---|---|
| **Registry** | `eaeraregistry.azurecr.io` | Địa chỉ registry lưu trữ image |
| **Image name** | `backend/crm-api` | Tên image (có thể kèm namespace/path) |
| **Tag** | `v2.1` | Phiên bản image |

> Thuật ngữ chuẩn của Docker/OCI là **FQIN** (*Fully Qualified Image Name*) hoặc *image reference*; "FQCN" là cách dùng trong khóa học, không phải thuật ngữ chuẩn thống nhất *(bổ sung từ nghiên cứu ngoài video)*.

**Quyền truy cập registry:**

- Chỉ dùng image → thường miễn phí, không cần tài khoản
- Cần login (Docker Hub) khi: upload image riêng lên registry, dùng các DevOps features bổ sung

**Demo Docker Hub — đánh giá độ tin cậy image:**

- Tìm image bằng CLI: `docker search nginx` — liệt kê mọi image có "nginx" trong tên hoặc mô tả; official build nằm trên cùng *(lệnh trong lab cuối bài)*
- Ai cũng upload được image lên Docker Hub → phải biết đánh giá nguồn gốc. Các nhãn tin cậy:
  - **Docker Official Image** — Docker đã review, tin cậy nhất
  - **Verified Publisher** / **Sponsored OSS** — các nhãn tin cậy khác
- Số **pulls** cũng là chỉ số độ phổ biến (nginx official > 1 tỷ pulls) *(transcript chép nhầm thành "image pools")*
- Click vào image → trang chi tiết có mô tả + cách dùng. Mẹo: không biết chạy ứng dụng thế nào → mở trang image trên Docker Hub hoặc registry đang host nó

**Vòng đời image qua registry** *(bổ sung từ nghiên cứu ngoài video)*:

```
build → push → registry (lưu theo repository + tag) → pull → run
```

| Bước | Lệnh |
|---|---|
| Build image | `docker build -t my-app:v1 .` |
| Đăng nhập registry | `docker login` |
| Push image lên registry | `docker push username/my-app:v1` |
| Pull image về máy | `docker pull username/my-app:v1` |

- Kubernetes Node / Docker Host pull image từ registry; container runtime dùng image đó để chạy container
- Trong CI/CD: pipeline tự động build image sau khi merge code → push lên registry → triển khai image đó lên Kubernetes

**Các registry phổ biến:**

| Registry | Đặc điểm |
|---|---|
| **Docker Hub** | Phổ biến nhất; public + private repositories |
| **GHCR** (GitHub Container Registry) | Gắn với GitHub repositories và workflows |
| **ACR** (Azure Container Registry) | Của Microsoft Azure, thường dùng với AKS |
| **ECR** (Elastic Container Registry) | Của AWS |

**Phân biệt 4 khái niệm** *(tổng hợp cả bài)*:

| Khái niệm | Là gì |
|---|---|
| **Container image** | Gói chứa ứng dụng + toàn bộ dependencies |
| **Container** | Instance đang chạy từ image |
| **Container runtime** | Công cụ tạo và chạy container |
| **Container registry** | Kho lưu trữ và phân phối image |

**Registry ≠ Git repository:**

- Git: chia sẻ **source code** để developer chỉnh sửa, phát triển tiếp
- Registry: chia sẻ **build artifacts** (image đã đóng gói) để chạy và triển khai — không thay thế Git

**FQCN trong manifest Kubernetes** *(nhìn trước — chi tiết ở module sau)*:

`image: myregistry.azurecr.io/my-app:v1` = `registry / tên image / tag`. Node sẽ pull image từ registry nếu image chưa có sẵn trên Node; dùng private registry cần cấu hình thêm (imagePullSecrets).

## Phần 3 — Khởi tạo và quản lý container

**Lệnh khởi chạy container phụ thuộc vào công cụ quản lý:**

| Công cụ | Lệnh | Ghi chú |
|---|---|---|
| **Docker** | `docker run nginx` | Chỉ cần tên image — Docker mặc định lấy image từ Docker Hub |
| **podman** | `podman run docker.io/library/nginx` | Bắt buộc dùng **FQCN** — podman không mặc định registry nào |
| **Kubernetes** | `kubectl run mynginx --image=docker.io/library/nginx` | FQCN không bắt buộc; không chỉ định registry → lấy từ Docker Hub hoặc Google Container Registry |

- Trong Kubernetes, khởi chạy container = khởi chạy **pod**, bằng `kubectl` — không dùng lệnh docker lẫn podman

**Docker vs podman** *(transcript chép nhầm thành "Popman")*:

- **Docker** (ra đời 2013): nhanh chóng thành de facto standard để chạy container qua Docker CLI
- **podman**: tích hợp vào RHEL 8 từ 2018, hiện phổ biến ngang Docker. CLI cung cấp toàn bộ chức năng của Docker CLI — thiếu chức năng nào của Docker thì cộng đồng podman coi đó là bug → chuyển từ Docker sang podman càng dễ càng tốt
- **Kubernetes là container manager riêng** → dùng kubectl, không dùng docker/podman

**Chuẩn bị lab (Ubuntu Desktop VM):**

- Ubuntu Desktop chạy được cả Kubernetes lẫn Docker trên giao diện Linux — "gần nhất với môi trường thi". Chi tiết lab environment ở Bài 4
- Các bước:
  1. `sudo apt install git` (có thể đã cài sẵn) — lời giải lab cài kèm vim; vim không bắt buộc nhưng **có sẵn trên kỳ thi CKAD**
  2. `git clone` repo khóa học `github.com/sandervanvugt/ckad` — bị hỏi username/password nghĩa là gõ sai URL (thường sai ở phần "sandervanvugt")
  3. Từ thư mục CKAD: chạy script `minikube-docker-setup.sh` (cài minikube, kèm luôn docker — chi tiết ở Bài 4)
  4. `sudo reboot` — không bắt buộc, nhưng đơn giản nhất nếu không muốn xử lý thủ công theo cách Linux
  5. Kiểm tra docker daemon: `sudo systemctl status docker` → cần thấy **active (running)**

**Demo — các lệnh Docker cơ bản:**

| Lệnh | Kết quả |
|---|---|
| `docker run nginx` | Fetch image nginx (web server phổ biến) từ Docker Hub về máy rồi chạy — nhưng output tràn màn hình (foreground) |
| `docker run -d nginx` | Chạy **detached mode** — không chiếm foreground, tiện quản lý và phân tích |
| `docker ps` | Liệt kê các container đang chạy |
| `docker run busybox` | Fetch busybox về nhưng container thoát ngay (xem dưới) |
| `docker run -it busybox` | Mở **interactive terminal** vào shell của container |

**Các cột trong output `docker ps`:**

- **CONTAINER ID** — ID số dùng để quản lý container tiếp theo
- **IMAGE** — image đã dùng để chạy
- **COMMAND** — entry point: lệnh mặc định mà image thực thi
- **CREATED / STATUS** — thời điểm tạo / đã chạy bao lâu
- **NAMES** — tên tự sinh

**Vì sao busybox không hiện trong `docker ps`:**

- Entry point mặc định của busybox là shell → shell không thấy standard out nên thoát ngay lập tức
- Exit code 0 — theo góc nhìn của lệnh mặc định thì "không có gì sai"; container chỉ thực sự chạy khi mở interactive shell (`-it`) hoặc truyền command làm tùy chọn dòng lệnh
- Container đã exit không hiện trong `docker ps`; cần `docker ps -a` để thấy nó (kèm trạng thái exited + exit code)

**Bên trong container (`docker run -it busybox`):**

- Prompt hash = shell của container; `whoami` → **root**
- Điểm mấu chốt: file system đang thấy **không phải** root file system của host OS — là root file system của **container image**, chạy trong namespace bị giới hạn hoàn toàn

## Phần 4 — Quản lý container

Quản lý container bằng Docker/podman **không nằm trong phạm vi đề CKAD** — học để hiểu chuyện gì xảy ra trong môi trường container standalone. Tác giả demo Docker vì minikube setup script cài sẵn Docker; mọi lệnh dùng được với podman như nhau. *(Muốn đào sâu: các khóa "Getting Started with Containers" / "Red Hat EX-188" của cùng tác giả.)*

**Các lệnh thiết yếu:**

| Lệnh | Chức năng |
|---|---|
| `docker ps` | Liệt kê container đang chạy |
| `docker ps -a` | Container đang chạy **+ toàn bộ container đã chạy trong quá khứ** |
| `docker images` | Các image đã fetch về máy |
| `docker run` | Tạo **container mới** từ image |
| `docker start` | Khởi động lại container **đã dừng** — đừng nhầm với `run` |
| `docker stop` | Dừng container đang chạy |
| `docker rm` | Xóa container |
| `docker inspect` | Xem toàn bộ thuộc tính của container — lệnh troubleshoot rất tiện |
| `docker kill` | Cut off container ngay — **không bao giờ dùng tay** (xem dưới) |

**Demo — vòng đời container:**

1. `docker run --name web nginx` (quên `-d` → chạy foreground) → Ctrl+C. Tên `web` **vẫn bị chiếm dụng** → muốn tạo container mới tên `web` phải remove (hoặc rename) cái cũ: `docker rm web`
2. Chạy lại `docker run -d --name web nginx` → output là container ID → container đang chạy; xác nhận bằng `docker ps`
3. **Nhiều container cùng chạy một image nginx** — trên Linux thường chỉ chạy được một process nginx; container chạy trong isolation hoàn toàn nên chạy song song thoải mái

**Đằng sau `docker ps -a`:**

- Chạy container = tạo một **environment** cho nó: một thư mục trên host OS chứa image + mọi thay đổi của container
- Container **stopped** → thư mục vẫn còn → vẫn hiện trong `docker ps -a`
- Container **removed** (`docker rm`) → thư mục bị xóa → không hiện nữa

**`docker images`:** nginx chỉ xuất hiện **một lần** dù có nhiều container đang chạy nó — image chỉ cần fetch một lần

**`docker stop` vs `docker kill`:**

- `docker stop web` → dừng đúng cách; sau đó `docker ps -a` thấy web "Exited ... seconds ago"
- `docker start web` → chạy lại container đã dừng
- `docker inspect web` → hiện "code behind the container": Path (file thực thi), Args (lệnh thật sự được chạy), toàn bộ thuộc tính Linux của container
- `docker kill` = cắt đứt ngay lập tức, không có gì được đóng đàng hoàng. **Không dùng tay lệnh này**: `docker stop` cho container vài giây để dừng tử tế; nếu không dừng được thì kill được phát tự động — nhưng ít nhất container đã có cơ hội đóng các file đang mở

## Phần 5 — Container logs

**Vấn đề:** container mặc định không kết nối vào standard out — chạy nền như service process trên máy tính thường. Vậy làm sao thấy output của ứng dụng chính?

**Giải pháp — container log:**

- Là file nằm trong **thư mục được tạo cho từng container** (môi trường của container, xem Phần 4), chứa application output: **stdout + stderr**
- Xem bằng `docker logs <tên-container>`

**Demo — troubleshoot container fail:**

1. `docker run --name mydb mariadb` — cố tình không dùng `-d` vì dự kiến fail; không detached nên thấy output hiện ngay trên màn hình rồi trả prompt
2. `docker ps` → không thấy; `docker ps -a` → `Exited (1)` — **exit code 1** = ứng dụng **không** dừng đúng cách (có lỗi) — đối lập với exit 0 của busybox ở Phần 3
3. Ứng dụng exit với code ≠ 0 → thường sinh logs → `docker logs mydb` lộ nguyên nhân: mariadb cần một **biến môi trường** để khởi động mà không được cung cấp
4. Biến đó có thể truyền bằng `--env` — nhưng theo tác giả đừng đi sâu: cần biết cách làm việc này trong **Kubernetes** mới quan trọng *(nhìn trước — module sau)*

> Quy trình troubleshoot chuẩn: container fail → `docker ps -a` xem exit code → `docker logs` tìm nguyên nhân.

## Tổng hợp lệnh Docker trong Bài 1

### Image & registry

| Lệnh | Ý nghĩa |
|---|---|
| `docker search nginx` | Tìm mọi image có "nginx" trong tên hoặc mô tả; official build nằm trên cùng |
| `docker images` | Liệt kê các image đã fetch về máy (image chỉ fetch 1 lần dù chạy nhiều container) |
| `docker login` | Đăng nhập registry (cần khi push image riêng hoặc dùng DevOps features) *(bổ sung từ nghiên cứu ngoài video)* |
| `docker build -t my-app:v1 .` | Build image từ Dockerfile, gán tên + tag *(bổ sung từ nghiên cứu ngoài video)* |
| `docker push username/my-app:v1` | Đẩy image lên registry *(bổ sung từ nghiên cứu ngoài video)* |
| `docker pull username/my-app:v1` | Tải image về máy *(bổ sung từ nghiên cứu ngoài video)* |

### Vòng đời container

| Lệnh | Ý nghĩa |
|---|---|
| `docker run nginx` | Tạo **container mới** từ image (fetch image trước nếu chưa có) — mặc định chạy foreground |
| `docker run -d nginx` | Như trên nhưng **detached mode** — chạy nền, không chiếm terminal |
| `docker run -it busybox` | Mở **interactive terminal** vào shell của container |
| `docker run --name web nginx` | Đặt **tên riêng** cho container (tên bị chiếm dụng cho tới khi `docker rm`) |
| `docker ps` | Liệt kê container **đang chạy** (ID, IMAGE, COMMAND, CREATED/STATUS, NAMES) |
| `docker ps -a` | Bao gồm cả container **đã exit** — kèm trạng thái + exit code |
| `docker start web` | Khởi động lại container **đã dừng** (không tạo mới — khác `run`) |
| `docker stop web` | Dừng container **đúng cách** — cho vài giây đóng file tử tế, tự kill nếu quá hạn |
| `docker rm web` | **Xóa** container (thư mục môi trường trên host bị xóa; container đang chạy phải stop trước) |
| `docker kill web` | Cắt đứt container **ngay lập tức** — không bao giờ dùng tay |

### Logs & troubleshoot

| Lệnh | Ý nghĩa |
|---|---|
| `docker logs mydb` | Xem log của container — file stdout + stderr nằm trong thư mục môi trường của container |
| `docker inspect web` | Toàn bộ thuộc tính container: Path, Args (lệnh thật sự chạy), thuộc tính Linux — troubleshoot |

> Mọi lệnh trên dùng y nguyên với **podman** (CLI tương thích Docker). Trong **Kubernetes** thì không dùng docker/podman — dùng `kubectl`.
>
> Muốn đào sâu Docker từ fresher đến senior (network, volume, Compose, bảo mật, internals, phỏng vấn): xem [docker-mastery.md](docker-mastery.md) *(tài liệu tham khảo ngoài phạm vi CKAD)*.