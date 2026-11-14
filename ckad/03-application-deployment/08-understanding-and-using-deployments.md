# Bài 8 — Understanding and Using Deployments

Bài đầu của Module 3 — miền **Application Deployment** (20% đề thi). Pod đơn lẻ
không đủ cho ứng dụng thật: cần **scalability**, **updates** không downtime và
**self-healing**. Deployment là câu trả lời tiêu chuẩn, cùng ba resource đặc
biệt: StatefulSet, DaemonSet và HorizontalPodAutoscaler.

## Lộ trình bài học

1. Chạy ứng dụng bằng Deployment
2. Labels, selectors và annotations
3. Deployment scalability
4. Cập nhật Deployment và rollout history
5. StatefulSet
6. DaemonSet
7. Autoscaling — HorizontalPodAutoscaler (HPA)
8. End-of-lesson lab

## Phần 1 — Chạy ứng dụng bằng Deployment

- Pod chạy bằng `kubectl run` là **standalone pod**: không scale được nhiều
  instance, và muốn ứng dụng luôn có mặt thì cần ai đó thay pod khi nó die.
- **Deployment** là cách tiêu chuẩn để chạy ứng dụng **stateless** cần
  scalability — pod fail hoặc node fail thì instance được thay thế tự động
  *(nhắc lại Bài 6 Phần 4 — muốn pod được thay thế khi fail thì đưa vào
  Deployment)*.
- Deployment **không trực tiếp tạo pod**. Nó tạo và quản lý một API resource
  riêng — **ReplicaSet** — và ReplicaSet là thứ sinh ra pods:

```text
Deployment ──tạo & quản lý──► ReplicaSet ──sinh & giữ số lượng──► Pods
```

- Chia hai mức này có lý do: ReplicaSet lo việc **giữ đúng số replicas**;
  Deployment lo việc **cập nhật** — đổi image hay cấu hình pod thì Deployment
  tạo ReplicaSet mới thay cái cũ (Phần 4). Đó là cơ chế của rolling update và
  rollback.
- Còn hai workload resource đặc biệt nữa trong bài này: **StatefulSet** (Phần 5,
  cho ứng dụng stateful) và **DaemonSet** (Phần 6, một pod trên mỗi node) —
  cùng với **autoscaling** (Phần 7).

## Phần 2 — Labels, selectors và annotations

### Labels là keo kết nối các object

- **Labels** là cặp key/value gắn vào mọi Kubernetes object — cơ chế **kết
  nối** các object với nhau: Deployment biết pods nào thuộc quyền quản, Service
  biết forward traffic tới pods nào *(nhìn trước — Service ở Module 4)*.
- Deployment khai báo **selector** (`selector.matchLabels`) — chỉ quản lý pods
  có label khớp. Label nằm ở pod template nên pods sinh ra tự mang label này.
- Lệnh tạo tự gán label khác nhau tùy resource:

| Lệnh | Label mặc định |
|---|---|
| `kubectl create deployment <tên> --image=nginx` | `app=<tên>` |
| `kubectl run <tên> --image=nginx` | `run=<tên>` |

### Lệnh làm việc với labels

| Lệnh | Chức năng |
|---|---|
| `kubectl get pods --show-labels` | Hiện labels của từng pod |
| `kubectl get pods -l app=<tên>` | Chỉ lấy object khớp label (`--selector`) |
| `kubectl label pod <pod> env=test` | Thêm label cho object đang có |
| `kubectl label pod <pod> env-` | Xóa label (thêm `-` sau key) |

- Label thêm **sau này** trên Deployment không truyền xuống pods — labels không
  "inherit": pod chỉ mang label có trong pod template lúc sinh.
- Gỡ label đang khớp selector khỏi một pod đang được quản lý → pod đó **thoát
  khỏi tầm nhìn** của ReplicaSet: ReplicaSet thấy thiếu pod → sinh pod thay
  thế; pod cũ thành **orphan** — vẫn chạy, nhưng không còn ai quản.

### Annotations

- **Annotations** cũng là cặp key/value trong metadata nhưng **không dùng để
  select** — chỗ ghi thông tin cho tool và con người: `kubectl` ghi
  `last-applied-configuration`, Deployment ghi số `revision` của mỗi lần update
  (Phần 4).
- Dùng lệnh `kubectl annotate` giống hệt cách dùng `kubectl label`.

> **Ghi nhớ thi CKAD**: `--show-labels` và `-l key=value` là cách nhanh nhất để
> kiểm tra labels khi thi — nhanh hơn `describe`.

*(Transcript ghi "Surfaces" — chuẩn hóa thành "Services" khi nói về object kết
nối qua labels.)*

## Phần 3 — Deployment scalability

- **Scalability** = tăng/giảm số instance của ứng dụng — lý do cốt lõi để chạy
  Deployment thay pod đơn lẻ.
- Tạo deployment với số replicas sẵn: `kubectl create deployment scaledapp
  --image=nginx --replicas=3`. Đổi số replicas sau này:

```bash
kubectl scale deployment scaledapp --replicas=4
```

- Lệnh `scale` tác động lên **Deployment** — ReplicaSet tự tạo thêm pod (scale
  lên) hoặc xóa bớt pod (scale xuống) ngay lập tức.
- Tên pod do Deployment quản có dạng `<tên>-<hash-ReplicaSet>-<hash-pod>` —
  mỗi pod một suffix riêng, nhưng pods cùng một ReplicaSet chia đoạn hash giữa.
- Xóa thử một pod: ReplicaSet **sinh pod thay thế ngay** — đúng quy tắc đã học
  *(nhắc lại Bài 6 Phần 7 — xóa resource lower-level thì nó được tạo lại; muốn
  xóa thật thì xóa top-level)*.

### Demo

```bash
kubectl create deployment scaledapp --image=nginx --replicas=3
kubectl get all                # deployment + replicaset + 3 pods
kubectl scale deployment scaledapp --replicas=6
kubectl get pods               # đủ 6 pods
kubectl delete pod <pod>       # chọn 1 pod bất kỳ
kubectl get pods               # pod mới xuất hiện thay thế, tổng vẫn 6
```

**Verify:** sau khi xóa 1 pod, danh sách vẫn có 6 pods — pod bị xóa mất tên,
pod thay thế mang tên mới cùng hash ReplicaSet.

## Phần 4 — Cập nhật Deployment và rollout history

### Cơ chế rolling update

- Cập nhật Deployment = đổi gì đó trong **pod template** (image là trường hợp
  phổ biến nhất) → Deployment tạo **ReplicaSet MỚI** cho cấu hình mới; pods cũ
  được thay dần bằng pods mới — ứng dụng không bao giờ xuống hết cả cụm.
- **ReplicaSet cũ không bị xóa** — được scale về 0 và giữ lại. Đây chính là
  "lịch sử" cho phép **rollback**.
- Nhịp thay pod điều khiển bằng strategy **rollingUpdate**:

| Property | Ý nghĩa |
|---|---|
| `maxSurge` | Cho phép **thêm** tối đa bao nhiêu pod trong lúc update |
| `maxUnavailable` | Chấp nhận **thiếu** tối đa bao nhiêu pod so với replicas |

- Hai cách sửa image — kết quả như nhau:

```bash
kubectl set image deployment <tên> <container>=<image-mới>
kubectl edit deployment <tên>          # mở editor, sửa trực tiếp
```

### Rollout history và rollback

```bash
kubectl rollout history deployment <tên>              # danh sách revision
kubectl rollout history deployment <tên> --revision=2 # chi tiết 1 revision
kubectl rollout undo deployment <tên>                 # về revision trước
kubectl rollout undo deployment <tên> --to-revision=1 # về revision cụ thể
```

- Mỗi lần update Deployment tạo revision mới — cũng là con số trong annotation
  `deployment.kubernetes.io/revision` (Phần 2).
- Rollback = scale ReplicaSet cũ lên lại, ReplicaSet mới xuống — không tạo
  ReplicaSet nữa.

### Demo — rolling.yaml

File `rolling.yaml` trong course repo: deployment với strategy
`rollingUpdate` (`maxSurge`/`maxUnavailable`) và image nginx cũ bản 1.8.

```bash
kubectl apply -f rolling.yaml
kubectl set image deployment <tên> nginx=nginx:1.15
kubectl get pods                          # pods mới lên, pods cũ xuống dần
kubectl rollout history deployment <tên>
kubectl edit deployment <tên>             # sửa image thành 1.17
kubectl rollout undo deployment <tên> --to-revision=1
kubectl get replicasets                   # các ReplicaSet cũ vẫn còn, 0 replicas
```

- Rollout trong demo chạy **rất nhanh** — image nginx các bản này đã **cache**
  trên node, không phải pull. Muốn quan sát rõ tiến trình rolling, dùng image
  chưa có trên node.

*(Transcript ghi "max search" — chuẩn hóa thành `maxSurge`.)*

## Phần 5 — StatefulSet

### Khi nào Deployment không đủ

- Deployment sinh pods **thay thế lẫn nhau được** — hợp ứng dụng stateless.
- Database và ứng dụng **stateful** cần pods có **danh tính ổn định**: pod khởi
  động lại vẫn giữ đúng tên, đúng storage — không thể là "một pod vô danh
  trong cụm".
- **StatefulSet** cung cấp điều đó, và là cách chạy database "thủ công"; thực
  tế database thường cài qua Helm chart có sẵn *(nhìn trước — Bài 9 cài mysql
  chart sẽ gặp lại StatefulSet)*.

### Đặc điểm của StatefulSet

- **Tên pod ổn định và theo thứ tự**: `web-0`, `web-1` — tạo lần lượt (web-0
  Running rồi mới đến web-1), không phải hash ngẫu nhiên như Deployment.
- Yêu cầu một **headless Service** (Service không có ClusterIP) đi kèm — nền
  của network identity ổn định cho từng pod.
- **Storage ổn định qua `volumeClaimTemplates`**: mỗi pod tự có PVC riêng mang
  tên pod (vd `www-web-0`) → bind PV *(nhắc lại Bài 7 Phần 5–6 — PVC/PV)*.
  Pod bị xóa, pod sinh lại với cùng tên → **mount lại đúng PVC cũ**, dữ liệu
  còn nguyên.
- `storageClassName` trong volumeClaimTemplate:
  - Bỏ trống → dùng **default StorageClass** của cluster.
  - Cluster không có StorageClass nào → PVC chỉ bind được PV **provision sẵn**
    *(nhắc lại Bài 7 Phần 4 — storage class cần provisioner)*.
- **Xóa StatefulSet không xóa PVC** — dữ liệu được bảo vệ. Muốn dọn storage
  phải xóa PVC thủ công (`kubectl delete pvc www-web-0`). Khuyến nghị:
  **scale về 0 trước khi delete** StatefulSet để dọn sạch các pod.

### Demo — troubleshooting PVC không bind

**Hiện tượng:** apply `statefulset.yaml` → `kubectl get pods` → pod `web-0`
treo trạng thái **Pending**.

**Cách kiểm chứng:**

```bash
kubectl describe pod web-0
# Events: pod has unbound immediate PersistentVolumeClaims
```

**Nguyên nhân:** volumeClaimTemplate khai báo `storageClassName` **không tồn
tại** trên cluster → PVC `www-web-0` không bind được PV nào → pod không có
storage nên không schedule.

**Cách xử lý:**

1. Sửa file thành `statefulset-new.yaml`: **bỏ storageClassName** để dùng
   default StorageClass → apply.
2. Pod **vẫn Pending** — PVC cũ `www-web-0` vẫn còn với storageClassName sai:
   PVC đã tạo **không tự cập nhật** theo manifest mới.
3. `kubectl delete pvc www-web-0` → StatefulSet sinh PVC mới theo cấu hình
   mới → PVC **Bound** → pod `web-0` Running, rồi `web-1` theo sau.

**Kết luận:** sửa storageClass trong StatefulSet không chữa được PVC đã sinh —
phải xóa PVC cũ cho nó tạo lại.

## Phần 6 — DaemonSet

- **DaemonSet** chạy **một pod instance trên mỗi node** của cluster — node
  thêm vào thì pod mới xuất hiện trên node đó, node bỏ đi thì pod biến mất.
- Use case điển hình: **agent software** cần hiện diện trên mọi node — vd
  network plugin của cluster.
- DaemonSet là **API resource riêng** — không phải "Deployment chạy trên mọi
  node": Deployment đếm tổng số replicas, DaemonSet đếm **số node**.
- **Không có lệnh imperative** tạo DaemonSet → phải dùng YAML. Mẹo: mượn YAML
  của deployment rồi sửa:

```bash
kubectl create deployment daemonnginx --image=nginx \
  --dry-run=client -o yaml > daemon.yaml
```

Sửa `daemon.yaml` đúng 3 chỗ:

1. `kind: Deployment` → `kind: DaemonSet`
2. **Xóa dòng `replicas`** — số pod do số node quyết định
3. **Xóa block `strategy`** — DaemonSet không có strategy này

```bash
kubectl apply -f daemon.yaml
kubectl get ds,pods
```

- Minikube chỉ có **1 node** → demo thấy đúng 1 pod, không ấn tượng lắm. Trên
  cluster thật: pod có mặt trên **mọi node trừ control plane** — control plane
  mang taint chặn application pods thường *(nhìn trước — taints/tolerations ở
  module sau)*.

> **Ghi nhớ thi CKAD**: không có `kubectl create daemonset` — sinh YAML bằng
> `--dry-run=client -o yaml` từ deployment rồi sửa `kind`, xóa `replicas` và
> `strategy`.

*(Transcript ghi "daemon nginx" — chuẩn hóa thành `daemonnginx`; mô tả
"deployment starts one pod on every node" — chuẩn hóa: DaemonSet là API
resource riêng, không phải một loại Deployment.)*

## Phần 7 — Autoscaling — HorizontalPodAutoscaler (HPA)

- Phần 3 scale **thủ công**. **HorizontalPodAutoscaler (HPA)** làm việc đó
  tự động: thu thập resource usage của pods, so với ngưỡng đặt ra, rồi thêm/
  bớt replicas.
- Hai thành phần: **metrics-server** thu thập usage (addon trong Minikube) và
  **HorizontalPodAutoscaler** — API resource đọc metrics và quyết định scale.

**Cấu hình HPA + metrics-server không nằm trong đề CKAD** — trên kỳ thi chỉ
cần `kubectl scale` thủ công. Phần này để hiểu cơ chế.

### Demo

Build ứng dụng stress (PHP tiêu CPU khi nhận request) từ thư mục `autoscaling`
trong course repo:

```bash
minikube addons enable metrics-server
cd autoscaling
docker build -t php-apache .
kubectl apply -f hpa.yaml          # deployment + service php-apache
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl get hpa
```

- `--cpu-percent=50`: ngưỡng target 50% CPU; HPA được phép giữ từ 1 đến 10
  replicas.

Tạo tải và quan sát:

```bash
kubectl run -it load-generator --rm --image=busybox --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

- Chờ vài phút (metrics và workload cần thời gian) → `kubectl get hpa`:
  CPU **250%** so với ngưỡng 50% → HPA tăng replicas (demo: lên **5** pods —
  `kubectl get deployment php-apache` xác nhận).
- Dừng tải: `kubectl delete pod load-generator` → usage giảm → HPA scale
  **xuống chậm**, dần dần về 1 pod — không giảm một nhát.

*(Transcript ghi "--cpu% is 50" và "horizontal port autoscaler" — chuẩn hóa
thành `--cpu-percent=50` và HorizontalPodAutoscaler; "kubectl delete port
load generator" → `kubectl delete pod load-generator`; "minus mysql --rm" →
`--rm`; trong lệnh wget tác giả đọc nhầm `-1` — đúng là `wget -q`.)*

## Phần 8 — End-of-lesson lab

### Đề bài

Tạo deployment chạy image `nginx:1.9`, **5 replicas**, pods mang label
`type=proxy`, và khi update **không bao giờ thiếu quá 2 pods** so với số
replicas.

### Lời giải

**Bước 1 — sinh YAML:** deployment đặt tên `lab9deploy`:

```bash
kubectl create deployment lab9deploy --image=nginx:1.9 --replicas=5 \
  --dry-run=client -o yaml > lab9.yaml
```

**Bước 2 — thêm label:** sửa `lab9.yaml`, thêm `type: proxy` vào labels của
pod template.

**Bước 3 — strategy:** "không thiếu quá 2 pods" → `maxUnavailable: 2`. Không
lệnh imperative nào set strategy được → mượn từ một deployment dummy:

```bash
kubectl create deployment dummy --image=nginx
kubectl get deployment dummy -o yaml    # copy block strategy về lab9.yaml
```

Sửa trong `lab9.yaml` thành `maxUnavailable: 2`, rồi xóa deployment dummy.

**Bước 4 — apply và gặp lỗi:**

```bash
kubectl apply -f lab9.yaml
kubectl get pods                        # ImagePullBackOff
kubectl describe pod <pod>              # lỗi pull image
```

Image `nginx:1.9` quá cũ, registry không còn pull theo dạng này. Sửa bằng
`kubectl set image` (Phần 4) — đúng cách update deployment, không sửa file:

```bash
kubectl set image deployment lab9deploy nginx=nginx:latest
kubectl get pods --show-labels          # 5 pods Running + type=proxy
```

- Rollout diễn ra nhanh vì `nginx:latest` đã cache trên node — muốn thấy rõ
  hành vi rolling (thiếu tối đa 2 pods), dùng image chưa có trên node; kiểm
  tra image đã cache: `minikube ssh` → `crictl images`.

*(Transcript ghi "minus 0 yaml" — chuẩn hóa thành `-o yaml`; "image pool back
off"/"error image pool" → `ImagePullBackOff`/`ErrImagePull`.)*

## Tổng hợp lệnh Bài 8

| Lệnh | Chức năng |
|---|---|
| `kubectl create deployment <tên> --image=<img> [--replicas=N]` | Tạo deployment (imperative) |
| `kubectl scale deployment <tên> --replicas=N` | Đổi số replicas |
| `kubectl set image deployment <tên> <ctr>=<img>` | Đổi image → rolling update |
| `kubectl rollout history deployment <tên> [--revision=N]` | Xem lịch sử update |
| `kubectl rollout undo deployment <tên> [--to-revision=N]` | Rollback |
| `kubectl get rs` | Xem các ReplicaSet (cũ giữ 0 replicas) |
| `kubectl label/annotate <res> <tên> k=v` | Thêm label/annotation (`k-` để xóa) |
| `kubectl get <res> -l k=v` / `--show-labels` | Chọn theo label / hiện labels |
| `kubectl get ds` | Xem DaemonSet (1 pod mỗi node) |
| `kubectl autoscale deployment <tên> --cpu-percent=N --min=N --max=N` | Tạo HPA |

> Đến đây đã có đầy đủ công cụ chạy ứng dụng **đã đóng gói sẵn image**. Bài 9
> chuyển sang chuyện cài **cả bộ ứng dụng** vào cluster: Helm và Kustomize.
