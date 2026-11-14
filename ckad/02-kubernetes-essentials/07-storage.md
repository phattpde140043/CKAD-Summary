# Bài 7 — Storage

Bài học về storage trong Kubernetes: container storage mặc định là ephemeral;
pod volumes; và cách kết nối external storage một cách linh hoạt bằng
**persistent volumes (PV)**, **persistent volume claims (PVC)** và **storage
class** trong môi trường cloud-native.

## Lộ trình bài học

1. Container storage — ephemeral theo mặc định
2. Pod volumes và các volume type phổ biến
3. Persistent Volumes (PV) — API resource định nghĩa storage
4. Storage class & storage provisioner — tạo PV tự động
5. Persistent Volume Claims (PVC) — request storage và bind PV
6. Cấu hình pod storage bằng PV/PVC
7. End-of-lesson lab

## Phần 1 — Container storage mặc định là ephemeral

Khi container start, **container working environment** được tạo thành một
**directory trên host** chạy container. Trong directory này, một subdirectory
được tạo để **lưu các thay đổi bên trong container** — và subdirectory này là
**ephemeral**: nó **biến mất khi container biến mất**.

→ Container storage mặc định là ephemeral: file ghi bên trong container không
sống sót qua container lifecycle.

## Phần 2 — Pod volumes

### Pod volume là gì

- Pod volume được define là **property của pod** — thuộc pod API resource
  specification. Danh sách các loại storage address được: `pod.spec.volumes`.
- Container muốn dùng volume phải mount qua
  `pod.spec.containers.volumeMounts`.
- Pod volume hợp lý khi pod chạy trong môi trường có **storage type cụ thể** —
  môi trường dùng NFS → define pod volume loại NFS.
- Cần **flexibility** hơn → **persistent volumes** (Phần 3). Use case DevOps:
  ứng dụng đi qua pre-production rồi production, mỗi môi trường dùng storage
  type khác nhau → cần PV configured để pod **pick up storage tự động**.

> **Ghi nhớ thi CKAD**: không có lệnh imperative dễ tạo pod kèm volume — trên
> kỳ thi hãy dùng **documentation** ("configure a volume for a pod"),
> copy-paste example configuration.

### Các pod volume type phổ biến

| Type | Là gì | Đánh giá |
|---|---|---|
| `emptyDir` | Temporary directory trên host chạy pod | Ephemeral + **host-bound**; tiện **testing**; nhiều instance ứng dụng → không sync gì với nhau |
| `hostPath` | Trỏ persistent directory trên host chạy pod | Hơn emptyDir (sống sót qua container lifetime) nhưng vẫn host-bound — không hợp cloud environments |
| `persistentVolumeClaim` | Connect tới persistent volume available | Chi tiết từ Phần 3 trở đi |
| FC (Fibre Channel), iSCSI | Storage "thật" cho real life | Cần additional setup → **không nằm trong CKAD** |

### Demo — Pod với emptyDir volume (Redis)

Bắt đầu từ browser: vào documentation, tìm **"configure a volume for a pod"**
(tìm "storage" hay "volume" cũng ra thông tin — nhưng đây mới chính xác là thứ
cần dùng trên kỳ thi). Copy example YAML vào `podvolume.yaml`:

- Pod chạy **Redis**
- `spec.volumes`: volume tên `redis-storage`, type `emptyDir: {}`
- `spec.containers.volumeMounts`: mount `redis-storage` vào path `/data/redis`
  — nơi Redis ghi data (vốn đã ephemeral; dùng emptyDir thì vẫn ephemeral nên
  *"doesn't make too much sense"* cho ứng dụng này — chỉ để demo)

```bash
kubectl apply -f podvolume.yaml     # hoặc apply trực tiếp từ URL trên slide
kubectl describe pod redis | less
```

`describe` cho thấy 2 phần:

- **Mounts**: `/data/redis` from `redis-storage`
- **Volumes**: `redis-storage` — Type **EmptyDir**, không set gì thêm

Viết một file vào volume rồi xem nó thực sự nằm đâu:

```bash
kubectl exec -it redis -- touch /data/redis/hello-world
```

Vào Kubernetes host để hiểu storage được back thế nào trên máy chạy ứng dụng:

```bash
minikube ssh
crictl ps | grep redis                                  # container ID của redis
crictl stop $(crictl ps | awk '/redis/ {print $1}')     # "fancy Linux way"
```

- `$(...)` là **command substitution**: `crictl ps | awk '/redis/ {print $1}'`
  tự động tìm ID container redis rồi đưa cho `crictl stop`.
- Container bị kill → kubelet restart nó (default restart policy).

Thoát minikube, quay lại laptop kiểm tra:

```bash
kubectl get pods                                  # pod redis đã được restart
kubectl exec -it redis -- ls -l /data/redis      # hello-world VẪN CÒN
```

File sống sót qua container restart — volume thuộc về **pod**, không phải
container. Còn nằm ở đâu trên host?

```bash
minikube ssh
sudo find / -name hello-world 2>/dev/null
```

Kubelet đã tạo emptyDir storage trong một path "incredible" dài dưới
`/var/lib/kubelet/pods/<pod-ID>/volumes/` *(bổ sung từ nghiên cứu ngoài video:
path chuẩn theo pod — transcript nói "ID of the container" nhưng emptyDir được
quản lý theo pod)*. Path chính xác **không cần reproduce trên kỳ thi** — điều
quan trọng là hiểu storage nằm trên host thế nào.

Xóa pod và xem storage phản ứng ra sao:

```bash
kubectl delete --force pod redis
minikube ssh
sudo find / -name hello-world 2>/dev/null     # file đã BIẾN MẤT
```

EmptyDir storage gắn với vòng đời pod — xóa pod (và container nó quản lý) thì
không còn lý do giữ storage lại nữa.

> ⚠️ Trong demo tác giả dùng `--force` — chính ông gọi là *"the ugly --force
> option"*. Với pod đơn lẻ, `kubectl delete pod redis` thường đã đủ; về cảnh
> báo force delete nói chung, xem Bài 6 Phần 7. *(bổ sung từ nghiên cứu ngoài
> video)*

**Bottom line:** emptyDir hợp lý khi cần **lưu tạm** hoặc **share files giữa
các container trong multi-container pod**; trong real life đây không phải
storage solution tốt nhất.

*(Transcript ghi "deutsch-data-redis-hello-world", "crictl ps pipe org on
single quote slash Redis slash print $1", "to grade then dev null", "empty disk
storage" — chuẩn hóa thành `touch /data/redis/hello-world`, `crictl ps | awk
'/redis/ {print $1}'`, `2>/dev/null` và emptyDir; các lệnh get/exec sau khi
exit minikube bị ghi nhầm thành "crictl" — thực tế là `kubectl`.)*

## Phần 3 — Persistent Volumes (PV)

### PV và PVC là hai API resource riêng

- Pod volume có thể dùng persistent storage **type** — nhưng persistent
  storage type **không phải** persistent volume: **persistent volume (PV) là
  API resource riêng**, dùng để định nghĩa storage.
- Pod truy cập PV qua **persistent volume claim (PVC)** — cũng là API resource
  riêng.
- Lợi ích: **decoupling** — pod không nối vào storage type cụ thể; nó nhận
  storage **available** theo PVC đang dùng.

Quy trình:

1. Pod dùng **PVC** — PVC chỉ là một **request for storage**: *"hey, I need 10
   GiB of storage"*.
2. PVC kiểm tra trên network storage gì available.
3. PVC connect tới persistent volume đáp ứng request.

→ Rất hữu ích trong DevOps environments: pre-production có thể dùng storage
giá rẻ, production dùng corporate storage đắt tiền — **cùng một PVC**, pod ở
mỗi môi trường tự động nối với storage available tại môi trường đó.

### Tạo PV: không có lệnh imperative

- Không có cách dễ tạo PV từ command line → dùng documentation, tìm **"create
  a persistent volume"**.
- Trong nhiều môi trường, PV được tạo **tự động** bằng **storage class
  resource + automatic storage provisioner** — ứng dụng trong cloud hỏi *"I
  need 10 gigabytes of storage"*, cloud provider tự provision đúng storage type
  available trong cloud (và bắt đầu tính tiền).

### storageClassName

- PV manual phải set **storageClassName** — property dùng để PVC connect tới
  PV; cần **ở cả hai đầu** (PV và PVC).
- Dùng để **group** storage theo loại: `preprod` cho pre-production storage,
  `prod` cho production storage.
- PV do storage class tạo tự động cũng mang storage class name tương ứng.
- PVC request storage **phải chỉ định storageClassName** để bind đúng loại
  storage — nếu không sẽ xảy ra "unexpected results".

### Kiến trúc storage trong Kubernetes

```text
Cách 1 — pod volume trỏ thẳng storage (không đủ linh hoạt):
  pod (container + volumes) ──volume──► NFS / storage type cụ thể
                                        container mount và ghi ra external storage

Cách 2 — cloud environment (linh hoạt):
  pod ──dùng──► PVC ──► storage class ──► provisioner ──tạo──► PV (on demand)

  (hoặc: PVC bind thẳng PV có sẵn — cả hai phải cùng storageClassName)
```

- Trong cách 2, volume define trong pod **không phải storage trực tiếp** mà là
  **PVC**; pod → PVC → storage class → PV được tạo on demand.
- Storage class là **optional**: PVC có thể bind PV trực tiếp — nhưng khi đó
  cả PVC lẫn PV đều phải có **storageClassName** khớp nhau, không set thì bind
  không hoạt động.

### Demo — Tạo PV thủ công từ documentation

Docs tìm **"create a persistent volume"** — example này **hơi khó tìm**, nên
practice trước để trên kỳ thi lấy được example nhanh. Copy example vào
`pvdemo.yaml`:

| Property | Giá trị | Ý nghĩa |
|---|---|---|
| `metadata.name` | `task-pv-volume` | Tên PV |
| `spec.storageClassName` | `manual` | Group storage — để PVC bind vào |
| `spec.capacity.storage` | `10Gi` | Dung lượng |
| `spec.accessModes` | `ReadWriteOnce` | Một trong 3 access modes |
| `spec.hostPath.path` | `/mnt/data` | Storage type thực — directory trên host |

Ba **access modes**: `ReadWriteOnce` (common & pretty safe), `ReadOnlyMany`,
`ReadWriteMany`. Lưu ý "Once" trong `ReadWriteOnce` tính theo **node** —
nhiều pod trên cùng node vẫn dùng chung volume; K8s ≥ 1.29 thêm
`ReadWriteOncePod` (đúng một pod) *(bổ sung từ nghiên cứu ngoài video)*.

```bash
kubectl explain persistentvolume.spec | less
```

Liệt kê toàn bộ storage types address trực tiếp từ PV: `awsElasticBlockStore`,
`azureDisk`, `azureFile`... kèm mô tả nhỏ cho từng loại.

> **CKAD**: hostPath **đủ dùng** cho kỳ thi; real life thì không muốn dùng
> hostPath.

```bash
kubectl apply -f pvdemo.yaml
kubectl get pv                           # 10Gi · RWO · Available · manual
kubectl describe pv task-pv-volume       # Type: HostPath · Path: /mnt/data
```

- `kubectl get pv` → PV ở trạng thái **Available** — 10Gi, access mode RWO,
  storage class `manual`.
- `kubectl describe` → source Type **HostPath**, Path `/mnt/data`.
- Hiện PV mới chỉ là **reference tới storage chưa ai dùng** — cần PVC (Phần 5)
  để thực sự sử dụng nó.

*(Transcript ghi "pv.task.pv.volume" và "AWS Selesic block score" — chuẩn hóa
thành `pv task-pv-volume` và `awsElasticBlockStore`; access mode thứ ba bị ghi
là "read only" — chuẩn là `ReadOnlyMany`.)*

## Phần 4 — Storage class & storage provisioner

### Cách storage class hoạt động

- Storage class làm việc với **storage provisioner** để tạo **persistent
  volumes on demand**.
- Chạy ứng dụng trong public cloud: cloud có sẵn storage provisioner,
  Kubernetes storage class nói chuyện với nó → storage được cung cấp **tự
  động**.

### Storage provisioner không thuộc vanilla Kubernetes

- Provisioner **không phải một phần của vanilla Kubernetes** — do
  **ecosystem** cung cấp, có thể được tích hợp sẵn trong một Kubernetes
  distribution.
- Không cài thêm gì → nhiều khả năng **không có provisioner** → **không dùng
  được storage class**.
- Storage class là kiểu resource **cần một ứng dụng đang chạy phía sau**:
  provisioner chính là ứng dụng chạy trong Kubernetes, nói chuyện với
  site-specific storage để tạo storage on demand. Không có provisioner thì
  storage class **không làm gì cả**.

> **CKAD**: cấu hình storage class + provisioner **không nằm trong đề thi** —
> nhưng không biết đến nó thì không thể hiểu hoàn chỉnh cách storage hoạt
> động.

### Demo — Storage provisioner trên Minikube

- Minikube có **add-ons** — chương trình bổ sung tích hợp dễ dàng; nhiều
  add-on **disabled mặc định** vì mỗi add-on bật thêm sẽ tiêu tốn resource
  trên máy.
- Add-on **`storage-provisioner` được bật sẵn** → Minikube mặc định đã có
  storage provisioner. Xem danh sách: `minikube addons list`.

```bash
kubectl get storageclass                 # standard
kubectl describe storageclass standard   # provisioner + options
```

- Storage class `standard` dùng provisioner **minikube-hostpath** — *"not the
  most spectacular"*: tạo hostPath storage mỗi khi có request storage. Cho mục
  đích học tập là đủ.
- Provisioner là **một ứng dụng đang chạy đâu đó** trong cluster — điều quan
  trọng ở đây: nó hiện diện và sẵn sàng phục vụ.

*(Transcript ghi "kubectl-get-storage-class" và
"kubectl-describe-on-the-storage-class-standard" — chuẩn hóa thành `kubectl
get storageclass` và `kubectl describe storageclass standard`.)*

## Phần 5 — Persistent Volume Claims (PVC)

### PVC là gì

- PVC là **API resource riêng**, định nghĩa **request for storage**.
- Mục đích: bind storage do một PV cung cấp vào **site** mà **không cần quan
  tâm storage type cụ thể** — application developer không cần biết đang dùng
  storage nào, chỉ cần dùng storage available tại site.

### Các attribute của PVC

| Attribute | Ý nghĩa |
|---|---|
| `storageClassName` | Trỏ tới storage — hoặc đóng vai trò **selector label**: tìm PV có storageClassName khớp |
| `accessModes` | `ReadWriteOnce` · `ReadOnlyMany` · `ReadWriteMany` |
| `resources.requests.storage` | Dung lượng storage cần |

### Quy tắc bind

- PVC xin 1MiB → bind được với **mọi PV offering ≥ 1MiB**.
- Bind là **exclusive — all or nothing**: PVC cần 1MiB mà bind vào PV 1TB →
  dùng **nguyên cả TB**, không còn phần nào cho PVC khác.
- **Không set storageClassName** → PVC chỉ bind với PV **do storage class tạo
  ra** (vì storage class là default storage provider).

### Demo — PVC bind vào PV manual

Vẫn trong trang docs "create a persistent volume", scroll xuống phần **create
a persistent volume claim** — copy example vào `pvcdemo.yaml`:

- name: `task-pv-claim`
- `storageClassName: manual` — **chính property này bind vào PV** (PV ở Phần 3
  cũng đang là `manual`)
- `accessModes: ReadWriteOnce` — sẽ tìm PV có access mode này
- `resources.requests.storage: 3Gi` — *G* hoa, *i* thường: Gi = gibibytes

```bash
kubectl apply -f pvcdemo.yaml
kubectl get pvc,pv
```

- PVC **Bound** — nhưng cột CAPACITY là **10Gi**, không phải 3Gi: PV offer
  10Gi và PVC theo kiểu **all or nothing** — chỉ cần 3 nhưng dùng trọn 10.

### Demo — Bỏ storageClassName thì sao?

Sửa `pvcdemo.yaml`: **xóa storageClassName**, đổi name, apply lại rồi `kubectl
get pvc,pv`:

- PVC bind vào PV tên `pvc-<uuid>` mang storage class **`standard`** — PV này
  do storage class trong Minikube **tạo tự động** (dynamic provisioning vào
  cuộc).
- Nếu **không có storage class** available **và** PVC không có
  storageClassName → **không có bind nào xảy ra — storage dysfunctional**.

*(Transcript ghi "G little bit I" — chuẩn là `Gi`; lệnh get bị gõ nhầm "pvc,
pc" (priority class) rồi tự sửa thành `kubectl get pvc,pv`.)*

## Phần 6 — Cấu hình pod storage bằng PV/PVC

Đã có PV + PVC — việc cuối cùng: **nối pod vào chuỗi**.

- Trong pod, define một volume có type **`persistentVolumeClaim`**; volume này
  mount vào container qua `volumeMounts` như mọi volume khác.
- PVC là resource riêng (đã tạo ở Phần 5). Khi chạy ứng dụng bằng YAML, thường
  **đặt YAML ứng dụng + YAML PVC cạnh nhau** — gom chung là hợp lý.

### Demo — Pod dùng PVC

Cũng trong trang docs đó (task "configure a pod to use a persistent volume for
storage"), lấy phần **create a pod** → copy vào `podpvc.yaml`:

- Volume tên nội bộ `task-pv-storage`, type **persistentVolumeClaim**,
  `claimName: task-pv-claim` — thay vì trỏ thẳng vào một storage cụ thể, volume
  trỏ vào PVC
- `volumeMounts.mountPath: /usr/share/nginx/html` — thứ PVC cung cấp sẽ được
  mount tại đây

```bash
kubectl apply -f podpvc.yaml
kubectl describe pod task-pv-pod
```

- `describe` → Volumes: `task-pv-storage` đến từ một persistent volume claim —
  **pod chỉ biết claim name**, không biết gì về storage phía sau.

Kiểm chứng data thực sự xuống storage:

```bash
kubectl exec task-pv-pod -- touch /usr/share/nginx/html/testfile
kubectl describe pv task-pv-volume      # Type: HostPath · Path: /mnt/data
minikube ssh
ls /mnt/data                            # testfile nằm ở đây
```

Chuỗi đầy đủ: mount point trong pod → PVC → PV → directory trên host. Với
hostPath thì dễ tìm file; storage type khác không phải lúc nào cũng dễ vậy.

*(Transcript ghi nhất quán "pot" cho "pod" — chuẩn hóa: `podpvc.yaml`,
`task-pv-pod`; "user share nginx.html" → `/usr/share/nginx/html`.)*

## Phần 7 — End-of-lesson lab

### Đề bài

1. Tạo **persistent volume**: storage type **hostPath**, directory
   **`/storage`**, access mode **ReadWriteMany**.
2. Chạy pod image **busybox**, default command **`sleep infinity`**.
3. Pod mount PV storage vào directory **`/data`**.
4. Dùng **`kubectl cp`** copy file **`/etc/hosts`** từ máy vào PV storage;
   verify trên storage host rằng storage đã được ghi.

### Lời giải

Làm hết trong **cùng một trang docs** — "configure a pod to use a persistent
volume for storage" chứa mọi thứ cần. Copy lại example **từ đầu**, đừng tái
dùng file cũ — *"you won't have these files on the exam as well"*.

**Bước 1 — PV** (`lab7pv.yaml`): name `lab7pv` · storageClassName `manual`
(giữ nguyên) · capacity giữ nguyên · accessModes đổi thành **ReadWriteMany** ·
hostPath đổi thành **`/storage`**:

```bash
kubectl apply -f lab7pv.yaml
```

**Bước 2 — PVC** (`lab7pvc.yaml`): copy PVC example → name `lab7pvc` ·
accessModes đổi thành **ReadWriteMany** (khớp PV):

```bash
kubectl apply -f lab7pvc.yaml
kubectl get pv,pvc        # lab7pv BOUND to lab7pvc
```

- Claim hiển thị `default/lab7pvc` — `default` là **namespace**, không liên
  quan gì đến storage class `standard` đang chạy trong Minikube.

**Bước 3 — Pod** (`lab7pod.yaml`): copy pod example → name `lab7pod` · volume
tên nội bộ giữ nguyên `task-pv-storage` (chỉ dùng nội bộ) · image đổi thành
**busybox** · ports bỏ qua · volumeMounts giữ `task-pv-storage` · trong volume
đổi `claimName` thành **`lab7pvc`**.

Syntax `sleep infinity` khó nhớ? Generate thử để xem Kubernetes hiểu nó thế
nào:

```bash
kubectl run dummy --image=busybox --dry-run=client -o yaml -- sleep infinity
# → args: ["sleep", "infinity"]
```

→ thêm `args: ["sleep", "infinity"]` vào pod, rồi apply. Trong demo, tác giả
apply xong mới phát hiện **quên đổi mountPath** (vẫn `/usr/share/nginx/html`)
→ `kubectl delete -f lab7pod.yaml`, sửa mountPath thành **`/data`**, apply lại
— bài học: quá tập trung vào syntax command dễ bỏ sót yêu cầu khác của đề.

**Bước 4 — `kubectl cp`** — *"one of these little surprises"*, lệnh có thể
chưa từng dùng → `kubectl cp -h`:

- Ghi chú quan trọng: **requires `tar`** present (kubectl cp hoạt động nhờ
  tar).
- Example: copy `/tmp/foo` local → `/tmp/bar` trong remote pod → copy vào pod
  là cách đưa file vào PV:

```bash
kubectl cp /etc/hosts lab7pod:/data
```

**Verify trên storage host:**

```bash
kubectl describe pv lab7pv       # hostPath /storage
minikube ssh
ls -l /storage                   # thấy file hosts — lab thành công
```

*(Transcript ghi "hostpad" → hostPath; "lab7pult" → `lab7pod.yaml`;
"read-write many" → `ReadWriteMany`; "kubectl cp ETC hosts lab7pod colon slash
data" → `kubectl cp /etc/hosts lab7pod:/data`; "MNT data" → `/mnt/data`.)*
