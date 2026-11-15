# Bài 13 — Deploying Applications the DevOps Way

Bài thứ hai của Module 5 — các **chiến lược DevOps** nâng cấp ứng dụng với
**zero downtime**: DevOps/GitOps làm nền, rồi hai chiến lược cụ thể —
**blue-green** và **canary** — mỗi chiến lược đều hiện thực được bằng Service
hoặc bằng Ingress.

## Lộ trình bài học

1. DevOps and GitOps
2. Blue-green deployments
3. Canary deployments
4. End-of-lesson lab

## Phần 1 — DevOps and GitOps

- Làm việc với Kubernetes, **DevOps** tập trung vào vài điểm:
  - **Configuration as code** — hiện thực bằng YAML files.
  - **Methodology dễ tái lập** — cùng một cách làm chạy lại cho kết quả như
    nhau.
  - **Continuous access to applications** — ứng dụng luôn reachable.
  - **Zero downtime application updates**.
- **GitOps** đưa automation lên cấp cao hơn: DevOps kết hợp **Git
  repositories**. YAML files được cung cấp từ một Git repo; **GitOps
  operator** — một quy trình tự động — pick up các thay đổi và apply vào
  Kubernetes cluster mà không cần thao tác tay.
- Đi cùng DevOps/GitOps, việc dùng đúng **strategy** càng quan trọng — hai
  strategy trong bài: **blue-green deployment** và **canary deployment**.

## Phần 2 — Blue-green deployments

- **Blue-green** là cách làm để ứng dụng được **upgrade an toàn sang bản
  mới**: bản mới đã có thể được **test sẵn trong khi bản cũ vẫn đang phục vụ
  users**. Đây là **methodology** — không gắn cứng vào lệnh hay option của
  phần mềm nào.
- Nhiều cách hiện thực: **Services**, **Ingress**, hoặc resources nâng cao
  như Istio.

| Cách | Đánh giá |
|---|---|
| **Service-based** | Diễn ra ở **lower level** — existing client connections có thể bị ảnh hưởng vì service không hề biết về các connection đang mở |
| **Ingress-based** | Hỗ trợ tốt existing connections — nhưng chỉ dùng được cho ứng dụng **HTTP/HTTPS** |

### Demo — Ingress-based blue-green (kustomize)

Thư mục `kustomize/blue-green` trong course Git repo, gồm **blue
customization** và **green customization**; xem `kustomization.yaml` là cách
kiểm tra nhanh nhất bên trong có gì:

| File | Nội dung |
|---|---|
| `deployment.yaml` | Deployment đơn giản, 1 replica, tên `blue-deploy` |
| `blue-svc` | Service NodePort expose blue deployment |
| `blue-cm` | ConfigMap tạo `index.html` hiển thị chữ **blue** |
| `myapp-ing` | Ingress — virtual hostname `myapp.local` trỏ tới blue service |
| patch | Patch blue deployment: chỉ chạm các property cần đổi — volume dùng ConfigMap mount vào `/usr/share/nginx/html` — cách cung cấp ConfigMap cho ứng dụng đã có sẵn |

**Bước 1 — chạy bản blue:**

```bash
kubectl apply -k .
kubectl get deploy,pods,svc,cm,ing
```

- `apply -k` báo **deprecation message** — giảng viên: tạm ổn, chưa phải lo.
- Dùng **danh sách phẩy** thay vì `kubectl get all`: `get all` không hiện
  hết resource — ConfigMap, Ingress, PV/PVC đều không có mặt.

**Bước 2 — truy cập bản blue:** thêm `myapp.local` vào `/etc/hosts` (`sudo
vim /etc/hosts`, tận dụng lại entry từ ingress demo trước đó), rồi `curl
myapp.local` → hiện **blue**. Đến đây hoàn thành bước 1–2 của quy trình.

**Bước 3 — tạo bản green và test:** green customization có mọi thứ **trừ**
`myapp-ing` — ingress là **central access point** của users; chưa có ingress
nào dẫn tới green nên users vẫn đang ở trên blue:

```bash
kubectl apply -k .        # trong thư mục green — giai đoạn test bản mới
```

**Bước 4 — switch:**

```bash
kubectl edit ing myapp    # service name: blue-svc → green-svc (không còn gì khác cần đổi)
curl myapp.local          # → green
```

- Ingress điều traffic tới một **service** — chỉ một thay đổi nhỏ là service
  đích từ blue thành green, toàn bộ traffic chuyển bên ngay lập tức.
- Khi mọi thứ đã ổn: `kubectl scale deploy blue-deploy --replicas=0`. Các
  resource blue **vẫn còn đó**, chỉ không còn ai dùng — giữ lại để dễ quay
  lại nếu cần.

### Demo — Service-based blue-green

**Bước 1 — bản blue, service đặt tên riêng:**

```bash
kubectl create deploy blue-nginx --image=nginx:1.14 --replicas=3
kubectl expose deploy blue-nginx --port=80 --name=bg-nginx
```

**Bước 2 — sinh manifest bản green từ blue:**

```bash
kubectl get deploy blue-nginx -o yaml > green-nginx.yaml
```

- Dọn file như thường lệ: bỏ runtime info trong `metadata`, cắt từ `status`
  trở xuống *(nhắc lại Bài 12 Phần 2)*.
- Dùng vi substitution đổi **blue** → **green** toàn file, và nên đổi cả
  image version sang bản mới (demo này không dùng ConfigMap — giữ đơn giản).

**Bước 3 — tạo bản green, test song song:**

```bash
kubectl create -f green-nginx.yaml
kubectl get pods           # blue-nginx và green-nginx cùng chạy
```

- Service `bg-nginx` là **entry point** của users — trực tiếp (ít xảy ra)
  hoặc qua ingress đứng trước (khả dĩ hơn). Muốn test từ ngoài thì có thể
  đổi service sang NodePort. Hiện tại service vẫn gửi traffic tới blue.

**Bước 4 — switch bằng service:**

```bash
kubectl delete svc bg-nginx; kubectl expose deploy green-nginx --port=80 --name=bg-nginx
kubectl delete deploy blue-nginx
```

- Mấu chốt: **xóa service cũ** (trỏ blue) rồi **lập tức tạo service mới cùng
  tên** (trỏ green) — dấu `;` chạy hai lệnh sát nhau, thời gian trống tối
  thiểu. Xong thì xóa deployment blue cũ.
- Nhận định của giảng viên: giữa hai cách, **ingress-based đẹp hơn** và tốt
  cho users hơn — đảm bảo connections không bị cắt.

> **Ghi nhớ thi CKAD**: blue-green = bản mới chạy **song song** + test trước,
> rồi **switch** (đổi service name trong ingress, hoặc delete + re-create
> service cùng tên). `kubectl get all` thiếu ConfigMap/Ingress/PV/PVC —
> liệt kê rõ: `kubectl get deploy,pods,svc,cm,ing`.

## Phần 3 — Canary deployments

- **Canary** expose bản mới cho **một số lượng users giới hạn** trước khi
  hoàn tất migration: đang chạy 3 replicas bản cũ → đưa vào **1 pod bản
  mới**, traffic load balance trên tất cả các pod. Chỉ thiểu số users dính
  bản mới nên **rủi ro tối thiểu**; có trục trặc thì gỡ pod mới là quay về
  tình trạng cũ.
- Hai cách hiện thực below, kèm các giải pháp ecosystem khác (như Istio ở
  Phần 2) cũng cung cấp sẵn — đây là **strategy**, mỗi nền tảng hiện thực
  theo cách riêng. Lời khuyên của giảng viên: hiểu cả hai.

| Cách | Đặc điểm |
|---|---|
| **Service-based** | Dễ hơn — dùng **common selector label** chung trên ứng dụng cũ lẫn mới |
| **Ingress-based** | Được ưu tiên — application nhận thay đổi **không cần reconnect**; dùng **hai Ingress resources** trỏ về cùng một virtual host |

### Cách hoạt động — Ingress-based canary

- Hai deployment (old + new), mỗi deployment có service riêng; trên cùng là
  **hai Ingress**: ing old và ing new.
- Cả hai Ingress dùng chung **host** (`myapp.info`) — phân biệt traffic bằng
  **annotations** đặt trên Ingress của bản mới:
  - `canary: "true"` — bật canary.
  - `canary-weight: "10"` — bản mới nhận 10% traffic.
- Ingress bản chất chỉ là **một bộ rules** do ingress controller xử lý — hai
  Ingress cùng host không xung đột: controller **merge** hai bộ rules và
  chia traffic đúng tỷ lệ.

### Demo — Ingress-based canary

**Bước 1 — hai ConfigMap old/new:**

```bash
echo new-version > index.html
kubectl create cm new-version --from-file=index.html
echo old-version > index.html
kubectl create cm old-version --from-file=index.html
```

- Lệnh echo thứ hai **ghi đè** `index.html` vừa tạo — không sao: file chỉ là
  input, nội dung bản đầu đã nằm sẵn trong ConfigMap.

**Bước 2 — bản cũ:** `canary.yaml` (course Git repo): deployment
`old-nginx`, 3 replicas, mount ConfigMap vào `/usr/share/nginx/html`:

```bash
kubectl apply -f canary.yaml
kubectl expose deploy old-nginx --port=80 --type=NodePort
```

**Bước 3 — bản mới từ chính file đó:** đổi **old** → **new** khắp
`canary.yaml` (một dòng sed, hoặc mở editor sửa tay — đều được), rồi:

```bash
kubectl apply -f canary.yaml
kubectl expose deploy new-nginx --port=80 --type=NodePort
```

- `--type=NodePort` không bắt buộc — nhưng tiện khi muốn test nhanh service
  mới đã available chưa.

**Bước 4 — Ingress bản cũ:** thêm host vào `/etc/hosts`, rồi *(nhắc lại
Bài 11 — tạo Ingress bằng lệnh)*:

```bash
kubectl create ingress old-version --rule=myapp.info/=old-nginx:80
curl myapp.info            # → old version
```

- Mọi request về old là bình thường — Ingress này đang redirect tất cả sang
  old-nginx, bản mới chưa tham gia.

**Bước 5 — Ingress canary cho bản mới:** file `new-ing.yaml`: Ingress
`new-version` trỏ tới `new-nginx`, **cùng host** `myapp.info`, thêm
annotations:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

```bash
kubectl apply -f new-ing.yaml
curl myapp.info            # lặp nhiều lần
```

- Kết quả: **đa số** request về old version, **thiểu số** về new version.
  Weight 10% vận hành theo round-robin nên **không bao giờ đúng chính xác
  10%** — nhưng tỷ lệ rõ ràng nghiêng về bản cũ.

*(bổ sung từ nghiên cứu ngoài video)*: tên đầy đủ của hai annotations là
`nginx.ingress.kubernetes.io/canary` và
`nginx.ingress.kubernetes.io/canary-weight` — transcript chỉ đọc "canary =
true" và "canary weight = 10".

*(Transcript ghi "canary await", "dapp.info" và "create-ing" — chuẩn hóa
thành canary-weight, myapp.info (theo hình vẽ ở phần lý thuyết) và
`kubectl create ingress`.)*

### Cách hoạt động — Service-based canary

- Một **service** với selector chọn một **label chung** (ví dụ `type:
  canary`). Ở backend, pods của ứng dụng cũ **và** ứng dụng mới đều mang
  label đó → service nối tới pods của cả hai bên.
- Điều chỉnh tỷ lệ traffic bằng **replicas**: bản cũ scale lên 4, bản mới để
  1 → 80% traffic về cũ, 20% về mới.

### Demo — Service-based canary

**Bước 1 — hai ConfigMap:**

```bash
echo old-nginx > index.html
kubectl create cm old-version --from-file=index.html
echo new-nginx > index.html
kubectl create cm new-version --from-file=index.html
```

**Bước 2 — bản cũ với label chung:** `canary.yaml` (phiên bản service-based,
khác file ở demo trên): deployment `old-nginx`, 3 replicas, mount ConfigMap;
pods mang **hai labels** — `app: old-nginx` và **`type: canary`**:

```bash
kubectl apply -f canary.yaml
```

**Bước 3 — bản mới:** đổi old → new trong `canary.yaml` rồi apply —
deployment `new-nginx` ra đời, pods cũng mang `type: canary`.

**Bước 4 — service chọn theo label chung:**

```bash
kubectl expose deploy old-nginx --name=theapp --port=80 \
  --selector=type=canary --type=NodePort
kubectl get svc                          # lấy NodePort
curl 192.168.49.2:31372                  # <minikube-ip>:<nodeport> của demo
```

- `--name` không quan trọng, `--port=80` bắt buộc — **`--selector` mới là
  mấu chốt**: trông như service "thuộc về" old-nginx, nhưng deployment đứng
  trong lệnh expose không quyết định gì — service chọn **mọi pod** có label
  `type=canary`, tức pods của cả old lẫn new.
- Kết quả curl: load balancing trộn old/new, **đa số về bản cũ** vì bản cũ
  nhiều replicas hơn (3 cũ + 1 mới = 75/25).

**Bước 5 — đổi tỷ lệ bằng replicas:** `kubectl scale` từng deployment — ví
dụ bản cũ 4 pods + bản mới 1 pod = 80/20.

> **Ghi nhớ thi CKAD**: service-based canary = **một service với selector
> label chung** (`--selector=type=canary`) trên pods của cả hai deployment;
> tỷ lệ traffic đi theo **tỷ lệ replicas**. Round-robin không phải phép
> tính chính xác: 10–20 request có thể ra 60-40, curl cả nghìn lần mới tiệm
> cận 75-25.

*(Transcript ghi "alt nginx" và "create cm alt" — chuẩn hóa thành old-nginx
và old-version theo ngữ cảnh demo.)*

## Phần 4 — End-of-lesson lab — Deploying Applications the DevOps Way

### Đề bài

Chạy deployment tên `old-bird` trên image **nginx:1.18** và deployment tên
`new-bird` trên image **nginx:latest**. Expose hai deployment bằng một
NodePort service sao cho **~90% traffic tới bản cũ**, **10% tới bản mới**.

### Lời giải

**Bước 1 — sinh manifest bản cũ, gắn label chung:**

```bash
kubectl create deploy old-bird --image=nginx:1.18 \
  --dry-run=client -o yaml > old.yaml
kubectl apply -f old.yaml
```

- Giữa hai lệnh là bước edit `old.yaml`: thêm label `type: bird` ở **cả hai
  chỗ** — `metadata.labels` của deployment **và** pod template
  (`spec.template.metadata.labels`). Dùng `kubectl label` không đủ: nó chỉ
  gán cho deployment, trong khi service selector cần label nằm trên **pods**.

**Bước 2 — sinh bản mới từ chính file đó:** edit `old.yaml` đổi mọi **old**
thành **new** (vi đủ dùng; không tự tin với vi thì dùng editor khác — chuyện
không nằm ở vi), đổi image thành `nginx:latest`, rồi apply lần nữa:

```bash
kubectl apply -f old.yaml
kubectl get all                         # old-bird và new-bird cùng chạy
```

**Bước 3 — service chọn label chung:**

```bash
kubectl expose deploy old-bird --name=bird --port=80 \
  --selector=type=bird --type=NodePort
kubectl get svc                         # bird trên một NodePort
```

**Bước 4 — kiểm chứng service chạm cả hai bản:** `curl
<minikube-ip>:<nodeport>` → "Welcome to nginx" (demo lần này không tạo nội
dung riêng nên hai bản giống hệt nhau — hơi boring). Bằng chứng nằm ở
`kubectl describe svc bird`, mục **Endpoints**: **2 endpoints** — một pod
cũ, một pod mới.

**Bước 5 — chỉnh tỷ lệ 90/10:**

```bash
kubectl scale deploy old-bird --replicas=9
```

- `kubectl get pods`: 9 pods old-bird + 1 pod new-bird. `describe svc bird`
  giờ ghi "**3 endpoints** +7 more" — đủ 10, tức **90% traffic về bản cũ**.
  Đó là toàn bộ lab.

*(Transcript ghi "old bird"/"new bird" — viết thành old-bird/new-bird theo
quy ước đặt tên của course (blue-nginx, old-nginx…); "port 3033" — NodePort
thật 5 chữ số, lấy từ `kubectl get svc`.)*

## Tổng hợp lệnh Bài 13

| Lệnh | Chức năng |
|---|---|
| `kubectl apply -k <thư mục>` | Apply kustomization directory (blue/green customization) |
| `kubectl get deploy,pods,svc,cm,ing` | Liệt kê nhiều loại resource một lệnh — `get all` thiếu cm/ing/PV/PVC |
| `kubectl edit ing <tên>` | Sửa Ingress live — đổi service đích khi switch blue-green |
| `kubectl scale deploy <tên> --replicas=N` | Scale replicas: tắt bản cũ (= 0) hay chỉnh tỷ lệ canary (9:1 = 90/10) |
| `kubectl expose deploy <tên> --name=<tên> --port=80 --selector=<label> --type=NodePort` | Service chọn pods theo label chung — mấu chốt canary service-based |
| `kubectl create ingress <tên> --rule=<host>/<path>=<svc>:<port>` | Tạo Ingress bằng lệnh *(nhắc lại Bài 11)* |

> Blue-green và canary gói trọn tư duy DevOps của bài: nâng cấp **song
> song**, chuyển traffic **từng bước**, rollback dễ. Bài 14 — **Working
> With the API** — lột một lớp sâu hơn: mọi lệnh kubectl đều đang nói
> chuyện với Kubernetes API; bài đó chỉ cách làm việc trực tiếp với API
> bằng cURL, xử lý API deprecations và mở rộng API bằng CRD cùng operators.
