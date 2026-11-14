# Bài 5 — Understanding and Using Pods

Bài học về khái niệm cơ bản nhất trong Kubernetes: **Pod**. Đây là bài đầu
tiên thuộc phạm vi **trực tiếp liên quan kỳ thi CKAD** — mọi ứng dụng
Kubernetes đều chạy dưới dạng pod.

## Lộ trình bài học

1. Các lựa chọn để chạy ứng dụng Kubernetes
2. Pod là gì — và những gì **không** phải là pod
3. Làm việc với YAML file — nền tảng của cách làm việc với Kubernetes theo kiểu DevOps
4. Có nên chạy nhiều container trong một pod không — và nếu có thì chạy thế nào
5. Tạo môi trường isolated cho pod bằng namespaces
6. Troubleshooting basics
7. End-of-lesson lab

## Phần 1 — Các lựa chọn chạy ứng dụng Kubernetes

### Pod — đơn vị nhỏ nhất

Pod là **minimal entity** để chạy ứng dụng Kubernetes: chạy container với đúng
một instance của container đó.

### Thông thường không chạy pod trực tiếp

- **Stateless application** → **Deployment**
  - Stateless = ứng dụng **không nhớ gì** về connection trước đó của user —
    mỗi request độc lập với request trước và sau. Web application thường stateless.
  - Deployment bổ sung: **scalability** (chạy nhiều instance của ứng dụng) và
    **zero-downtime application updates** dễ dàng.
- **Stateful application** (vd: database) → **StatefulSet**
  - Có thêm yêu cầu (additional requirements) → dùng phức tạp hơn Deployment.
  - Cách dễ để tạo StatefulSet: **Helm Package Manager** (học ở phần sau của
    khóa) — cài ứng dụng cùng toàn bộ dependencies cần thiết.

| Resource | Dùng cho | Đặc điểm |
|---|---|---|
| Pod | Chạy 1 container đơn lẻ | Minimal entity — đơn vị nhỏ nhất |
| Deployment | Stateless app | Scalability + zero-downtime updates |
| StatefulSet | Stateful app (vd: database) | Additional requirements → phức tạp hơn |
| Helm | Cài app kèm dependencies | Cách dễ tạo StatefulSet; học sau trong khóa |

> **Ghi nhớ thi CKAD** *(bổ sung từ nghiên cứu ngoài video)*: transcript nhấn
> mạnh *"normally, you don't run pods"* — nhưng đề thi CKAD vẫn thường yêu cầu
> tạo pod trực tiếp bằng `kubectl run`. Phân biệt rõ: pod = đơn vị cơ bản;
> Deployment/StatefulSet = cách quản lý pod có scalability & update strategy.

## Phần 2 — Pod là gì

### Kubernetes quản lý pod, không quản lý container

Pod là **minimal entity được Kubernetes quản lý**. Kubernetes **không quản lý
container** — nó quản lý pod, và từ một pod có thể khởi chạy **một hoặc nhiều
container**.

Vì sao cần pod? Pod **thêm Kubernetes properties** vào container — các thuộc
tính cho phép orchestrate container trong cluster/cloud environment:

- `nodeSelector` — chọn node nào sẽ chạy pod
- `priority` — gán priority cho pod
- `restartPolicy` — báo cluster làm gì nếu pod fail
- Pod còn có thể include **volumes** — cung cấp storage dễ dàng cho ứng dụng
  containerized

Muốn khám phá toàn bộ pod features:

```bash
kubectl explain pod.spec | less
```

Output rất dài nên pipe qua `less` — thấy các properties có thể set:
`activeDeadlineSeconds`, `automountServiceAccountToken`, `dnsConfig`... và
`kubectl explain` kèm giải thích cho từng feature.

### Standalone pod — nhược điểm

| Nhược điểm | Ý nghĩa |
|---|---|
| Không reschedule khi fail | Có sự cố với pod → pod **không** được restart |
| Không update strategy | Rolling update **không áp dụng** — chỉ bring down rồi bring up với setting mới → user đang connect sẽ bị downtime |
| Không scale | Pod thứ 2 là standalone pod **độc lập hoàn toàn** — không phải scale out |
| Không tự replace | Cluster không tự thay thế pod |

- Standalone pod vẫn được dùng phổ biến cho **testing & troubleshooting**.
- **Trên đề thi: nhiều task chính là về standalone pod** — *"forget about
  reality on the exam"*: đề yêu cầu chạy standalone pod thì chạy standalone pod.
- Ứng dụng thường trú (permanent) → **Deployment / StatefulSet**, vì chúng lo
  việc **availability** của các instance ứng dụng.

### Lệnh làm việc với standalone pod

```bash
kubectl run firstpod --image=nginx    # --image là argument bắt buộc
kubectl get pods                      # tổng quan pod đang chạy
kubectl describe pod firstpod | less  # toàn bộ properties của pod
```

- `--image` bắt buộc — thiếu nó `kubectl run` không biết chạy image nào.
- `kubectl describe` dùng được với **mọi resource**; trên pod cho thấy: **IP
  address** (pod giữ IP), container + state hiện tại (running/healthy), và
  **Events** ở cuối output — tổng kết mọi việc Kubernetes đã làm để chạy pod.

## Phần 3 — Làm việc với YAML: declarative vs imperative

### Configuration as code

Trong môi trường Kubernetes, DevOps được dùng nhiều — mục tiêu: deploy ứng dụng
**consistent & dễ reproduce**. Khi nhiều nhân viên IT phải chạy ứng dụng theo
đúng một cách, giải pháp là **configuration as code**: không gõ lệnh kèm thông
tin trên command line, mà viết **manifest file** mô tả cách chạy ứng dụng.

| Cách | Tên | Cách làm |
|---|---|---|
| Từ YAML file | **Declarative** | Tạo resource từ YAML manifest |
| Từ command line | **Imperative** | `kubectl run` cho pod đơn lẻ *(transcript ghi "individual bots" — đúng là "individual pods")*, `kubectl create` cho deployment |

- **Real life:** declarative được dùng rất nhiều — YAML để trong Git repo,
  Kubernetes operator chạy ứng dụng từ đó.
- **Trên kỳ thi:** không phải real life — chọn **cách nhanh nhất**, thường là
  imperative. Nhưng nhiều properties **không set được bằng imperative** → vẫn
  phải động đến YAML. Khóa học dạy cả hai: YAML khi hợp lý, imperative khi nhanh
  hơn.

### YAML là gì

- **Human readable data serialization language** — hoàn hảo cho DevOps tạo
  input file quản lý objects ở nhiều hệ thống: Kubernetes, Ansible...
- **Indentation thể hiện quan hệ** parent/child — nhận child object qua mức
  thụt lề của nó.
- Kết hợp với **Git** → cách xuất sắc quản lý & cập nhật configuration có cấu
  trúc.

### Thành phần của YAML manifest

Mọi thành phần YAML manifest trong Kubernetes đều là resource properties được
định nghĩa trong API:

| Field | Ý nghĩa | Ai viết |
|---|---|---|
| `apiVersion` | Version của API dùng cho object này | Người dùng |
| `kind` | Loại object: Pod, Deployment... | Người dùng |
| `metadata` | Thông tin administrative của object | Người dùng |
| `spec` | Cấu hình cụ thể cách resource được dùng | Người dùng |
| `status` | Trạng thái hiện tại của resource | **Cluster tự thêm** cho resource đang chạy |

### pod.spec.containers — các property phổ biến

Trong pod specification có container specification — nơi định nghĩa cách chạy
container:

- `name` — tên container
- `image` — image container sẽ dùng
- `command` — lệnh container chạy
- `args` — arguments cho command
- `env` — environment variables

Tra cứu bằng `kubectl explain` với **định dạng dotted** (`spec` là child của
`pod`):

```bash
kubectl explain pod                   # top-level fields: apiVersion, kind, metadata, spec, status
kubectl explain pod.spec              # toàn bộ properties của pod specification
kubectl explain pod.spec.containers   # properties set được trên containers
```

### Chạy YAML file

| Lệnh | Hành vi | Nếu resource đã tồn tại |
|---|---|---|
| `kubectl create -f resource.yaml` | Tạo resource từ YAML | **Fail** → không thực dụng lắm |
| `kubectl apply -f resource.yaml` | Tạo resource từ YAML | **Update** property cần update; không cần thì không làm gì — không fail |
| `kubectl delete -f resource.yaml` | Xóa resource định nghĩa trong file | — |
| `kubectl replace -f resource.yaml` | Thay thế resource hiện tại bằng spec trong file | Ít dùng — *"you might never ever use it"* |

> **Thông điệp của tác giả:** đừng **viết** YAML file — viết tay mất nhiều công.
> Hãy **generate** YAML (Phần 4).

## Phần 4 — Generate YAML file

Thêm `--dry-run=client -o yaml` vào `kubectl run` hoặc `kubectl create`, rồi
redirect stdout vào file:

```bash
kubectl run mynginx --image=nginx --dry-run=client -o yaml > mynginx.yaml
kubectl apply -f mynginx.yaml
```

- `--dry-run=client` — **không tạo gì trên cluster**, chỉ sinh code
- `-o yaml` — xuất code ở định dạng YAML
- `> mynginx.yaml` — ghi ra file; chạy **không có redirect** trước để xem YAML
  in ra màn hình
- Trên kỳ thi còn có thể **copy-paste YAML từ kubernetes.io/docs**.

*(Transcript ghi "my.nginx" và "//dryrun is client" — chuẩn hóa thành `mynginx`
và `--dry-run=client`.)*

Trong YAML sinh ra, để ý `spec` chứa rất nhiều properties có thể set, và
`containers` bên trong cũng vậy — **đây là việc bạn sẽ làm rất nhiều trên kỳ
thi**: generate YAML rồi sửa property trong spec. Tra cứu property bằng
`kubectl explain pod.spec` / `kubectl explain pod.spec.containers`.

## Phần 5 — Multi-container pod

### Có nên dùng multi-container pod?

Trong đa số trường hợp: **không**. **One-container pod là standard** — pod một
container dễ build và maintain hơn.

Ví dụ điển hình: ứng dụng có **web server (front-end) + database (back-end)**.
Nghĩ đến việc chạy một pod gồm 2 container (web server + database)? Vấn đề: khi
có version mới của database, làm sao **update mà không bring down web server**?
→ lý do mạnh để dùng single-container pod:

- Mỗi pod một workload → set được **specific settings tốt cho đúng workload
  đó** (pod spec của web server và database cần properties khác nhau).
- Ứng dụng gồm nhiều container → dùng **microservice**: các pod được quản lý
  **độc lập** (independently managed), nối với nhau bằng resources do
  Kubernetes cung cấp. Elegant hơn — update một phần của microservice mà không
  ảnh hưởng phần khác.

### Ba pattern multi-container hợp lệ

| Pattern | Vai trò | Ví dụ |
|---|---|---|
| **Sidecar** | Enhance primary application | logging |
| **Ambassador** | Đại diện primary container ra bên ngoài | proxy |
| **Adapter** | Adapt traffic/data pattern cho khớp pattern của ứng dụng khác trong cluster | |

### Sidecar container — chi tiết

- Cung cấp **additional functionality** cho main container — chức năng mà nếu
  chạy ở pod riêng thì **không hợp lý**, vì hai bên gắn bó quá chặt: **logging,
  monitoring, synchronizing**.
- Bản chất: main container và sidecar **chia sẻ shared resources** để trao đổi
  thông tin.
- Ví dụ phổ biến: **Istio service mesh** — addition quen thuộc của Kubernetes,
  **inject sidecar container vào pod** cho traffic management (Istio chuyên
  advanced traffic management; truyền parameters cần thiết tới ứng dụng; thường
  dùng **shared volumes** — học kỹ hơn ở phần sau).
- Lưu ý: **Kubernetes version gần đây đã đổi mô tả/định nghĩa sidecar
  container** → demo sidecar nằm ở **Bài 6.2**, không phải bây giờ.

### Demo — multi-container pod đơn giản

File `multi-container.yaml` trong course Git repo: pod tên `multi-container`,
`containers` là **list** gồm 2 container:

1. busybox — image `busybox`, command `sleep 3600`
2. nginx — image `nginx`

```bash
kubectl apply -f multi-container.yaml
kubectl get pods multi-container
```

- Cột **READY `0/2`**: kỳ vọng 2 containers, 0 sẵn sàng — chờ một lúc rồi get
  lại → **`2/2`**.
- Pod này có ý nghĩa thực tế không? Không — và đó chính là điểm chính: đa số
  trường hợp multi-container **không hợp lý**, nhưng **làm được** — chỉ cần
  define nhiều container trong list `containers`.

*(Transcript ghi "used to adopt a traffic or data pattern" — đúng là "adapt".)*

## Phần 6 — Namespaces: môi trường isolated cho pod

### Namespace là gì

- Kubernetes namespace resource **tận dụng Linux kernel namespaces** để cung
  cấp **resource isolation**.
- Các namespace khác nhau giúp **tách strict tài nguyên của từng customer** →
  enable **multi-tenancy**.
- Namespace cũng là nơi **apply security-related settings**: **RBAC**
  (role-based access control — permissions) và **quota** (restrictions).

### Vì sao nên dùng namespace

Cài ứng dụng Kubernetes phức tạp trong **namespace riêng** → quản lý dễ hơn.
Ứng dụng phức tạp gồm **nhiều resources liên quan nhau**; nếu chạy tất cả trong
cùng một namespace, cuối cùng resources sẽ quá nhiều và trộn lẫn vào nhau, khó
quản lý.

### Lệnh làm việc với namespace

| Lệnh | Chức năng |
|---|---|
| `kubectl get pods -A` | Xem resources ở **tất cả** namespace — cột đầu tiên là namespace name |
| `kubectl create ns <tên>` | Tạo namespace — namespace phải **tồn tại trước** khi chạy resource trong nó |
| `kubectl run <tên> --image=... -n <namespace>` | Chạy pod trong namespace cụ thể |

### Demo — và một sai lầm đáng học

Tạo namespace tên `secret`, chạy pod nginx trong đó:

```bash
kubectl create ns secret
kubectl run secret --image=nginx -n secret
kubectl get pods            # KHÔNG thấy pod — get mặc định chỉ hiện namespace default
kubectl get pods -n secret  # thấy pod secret đang chạy
```

Trong demo, tác giả **gõ nhầm** một cách vô tình nhưng rất instructive:

```bash
kubectl run pod secretpod --image=nginx -n secret   # SAI
```

- `kubectl create` **yêu cầu resource type** (`create ns`, `create
  deployment`...), nhưng `kubectl run` **chỉ dành cho pod** — không ghi
  resource type.
- Hậu quả: pod được tạo với **tên `pod`**, còn `secretpod` bị hiểu là
  **argument** truyền vào pod → **CrashLoopBackOff**.
- Troubleshooting chính sai lầm này:
  - `kubectl describe -n secret pod pod` — event **Back-off restarting failed
    container**; application đã finished với **exit code 127**.
  - `kubectl logs -n secret pod` — pod chạy default application
    (`docker-entrypoint.sh`) và báo lỗi *secretpod not found* → xác nhận
    `secretpod` đã bị truyền thành argument cho ứng dụng.
- Dọn dẹp — chữ `pod` xuất hiện **hai lần** (resource type + tên pod):

```bash
kubectl delete -n secret pod pod
```

*(Transcript ghi "kubectl create NS" và "back off restarting field container" —
chuẩn hóa thành `kubectl create ns` và `Back-off restarting failed
container`.)*

## Phần 7 — Troubleshooting basics

### Quy trình troubleshooting 4 bước

| Bước | Lệnh | Dùng khi |
|---|---|---|
| 1 | `kubectl get pods` | Tổng quan trạng thái hiện tại của tất cả pod |
| 2 | `kubectl describe pod <tên>` | Thấy pod nào không ổn → xem thông tin chi tiết của pod đang fail |
| 3 | `kubectl logs <pod>` | Truy cập **output đã log của ứng dụng** — ứng dụng fail thường đã sinh error message |
| 4 | `kubectl exec -it <pod>` | Mở shell trên pod để phân tích — **chỉ hữu ích khi pod đang chạy**; pod không start được thì vô dụng |

### Demo — troubleshooting pod MariaDB

Chạy database thiếu môi trường cần thiết:

```bash
kubectl run mydb --image=mariadb
```

**Giai đoạn 1 — đang khởi động** (~30 giây):

- `kubectl get pods` → **ContainerCreating**.
- `kubectl describe pod mydb` → Events cho thấy đúng thứ tự Kubernetes làm
  việc: **scheduler gán pod cho cluster node** (việc đầu tiên phải xảy ra) →
  node **pull container image** (image phải hiện diện trên node chạy container).

**Giai đoạn 2 — phát hiện lỗi**:

- `kubectl get pods` → STATUS lỗi và **RESTARTS 3** — pod đã bị restart 3 lần.
- Nguyên nhân restart: **default restart policy** — ứng dụng fail thì
  Kubernetes start lại (nói kỹ hơn ở phần sau của khóa).
- `kubectl describe pod mydb`:
  - Event cuối (event mới nhất luôn nằm cuối): **Back-off restarting failed
    container**.
  - Section container: State **Waiting**, Reason **CrashLoopBackOff**; Last
    State **Terminated**, Reason **Error**, **Exit Code 1**.

### Exit code là gì

- Cách **generic để ứng dụng báo kết quả cho parent shell**.
- **Exit code 0** = thành công — từ góc nhìn ứng dụng, không có gì sai.
- **Exit code khác 0** = không hoàn thành successfully — số bất kỳ, thường là **1**.

### Tìm root cause và fix

```bash
kubectl logs mydb
```

Output của ứng dụng chứa error: **"you need to specify one of
MARIADB_ROOT_PASSWORD..."** — MariaDB không start được nếu thiếu root password.
Fix:

```bash
kubectl delete pod mydb
kubectl run --help | less        # tra cứu cách truyền environment variable
kubectl run mydb --image=mariadb --env MARIADB_ROOT_PASSWORD=password
kubectl get pods                 # mydb đang Running
```

- `--env` truyền environment variable cho pod — MariaDB cần
  `MARIADB_ROOT_PASSWORD` để start.
- Đây **không phải cách lý tưởng** để set root password — cách đúng là
  **ConfigMaps & Secrets** (phần sau của khóa học).

*(Transcript ghi "kubectl run minus hpipe less" — chuẩn hóa thành `kubectl run
--help | less`.)*

## Phần 8 — End-of-lesson lab

### Đề bài

Tạo **một YAML file** đáp ứng: (1) tạo namespace tên `microservice`; (2) tạo
pod tên `microdb` chạy image MariaDB trong namespace đó; (3) tạo resources
bằng phương pháp **declarative**.

### Lời giải

Generate 2 phần YAML bằng dry-run rồi ghép vào một file:

```bash
kubectl create ns microservice --dry-run=client -o yaml > lab5.yaml
kubectl run microdb --image=mariadb --env MARIADB_ROOT_PASSWORD=password \
  -n microservice --dry-run=client -o yaml >> lab5.yaml
```

- `>` (single redirect) — tạo file, **overwrite** nếu đã tồn tại.
- `>>` (double redirect) — **append** output vào cuối file: lệnh thứ 2 phải
  append để không đè mất phần namespace.
- Mở file kiểm tra: thấy 2 phần cấu hình khác nhau → thêm separator `---`
  (**đánh dấu bắt đầu một YAML document**) trên đầu **cả namespace definition
  lẫn pod definition**.

*(Nhờ `-n microservice`, YAML sinh ra kèm `metadata.namespace: microservice` —
bổ sung từ nghiên cứu ngoài video.)*

Apply và verify:

```bash
kubectl apply -f lab5.yaml
kubectl get pods -n microservice   # microdb đang Running
```

Kết quả: namespace `microservice` và pod `microdb` được tạo cùng nhau từ một
file — đúng tinh thần declarative.

*(Transcript ghi "kubectl create-ns" — chuẩn hóa thành `kubectl create ns`.)*
