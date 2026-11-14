# Bài 9 — Installing Applications

Bài cuối của Module 3. Đến nay mỗi resource đều tự viết YAML riêng — bài này
gồm ba cách cài **cả bộ ứng dụng** vào cluster: YAML thuần, **Helm** (package
manager của Kubernetes) và **Kustomize** (tính năng sẵn có của kubectl).

## Lộ trình bài học

1. Cài ứng dụng từ một YAML file
2. Helm là gì; cài Helm binary
3. Làm việc với Helm repositories và search
4. Cài đặt ứng dụng bằng Helm — quản lý values và upgrades
5. Kustomize
6. End-of-lesson lab

## Phần 1 — Cài ứng dụng từ một YAML file

- Một ứng dụng trên Kubernetes hiếm khi chỉ là một resource: thường gồm
  **Deployment + Service + ConfigMap/Secret + storage...** cùng làm việc với
  nhau.
- Cách thô sơ nhất: gộp tất cả vào **một YAML file**, ngăn cách các document
  bằng dòng `---` *(nhắc lại Bài 5 — multi-document YAML)* rồi
  `kubectl apply -f app.yaml`.
- Cách này **chạy được nhưng không portable**: giá trị đặc thù từng nơi —
  image registry nội bộ, số replicas, storage class, domain... — lẫn vào
  config chung của ứng dụng; mang sang môi trường khác phải sửa tay trong
  file.
- Hướng giải quyết: **tách config chung của ứng dụng khỏi giá trị đặc thù
  từng môi trường**. Hai công cụ trong bài làm được việc đó — Helm (Phần
  2–4) và Kustomize (Phần 5).

## Phần 2 — Helm là gì; cài Helm binary

### Helm = package manager của Kubernetes

- Hai thành phần:
  - **Helm tool** — binary chạy trên máy client; cài riêng, không có sẵn.
  - **Chart** — "gói phần mềm": chứa mô tả ứng dụng + các **template**
    (Kubernetes manifests có chỗ chèn giá trị).
- Chart lưu local hoặc trên remote **Helm repositories** — ai cũng publish
  chart được, giống ai cũng upload image lên Docker Hub *(nhắc lại Bài 1 —
  phải biết đánh giá độ tin cậy nguồn)*.

### Cài Helm binary

1. Mở `github.com/helm/helm` → **Releases**.
2. Chọn package theo platform — tác giả dùng bản **ARM64**; server/cloud
   thường là `linux-amd64`.
3. Giải nén và đưa binary vào PATH:

```bash
tar xvf helm-v*-linux-arm64.tar.gz
sudo mv linux-arm64/helm /usr/local/bin
helm version
```

- Bash completion: `source <(helm completion bash)`.

### Helm repository ở đâu

- Helm **không có repository mặc định** — muốn cài chart phải tự thêm repo
  vào client (Phần 3).
- Trang tra cứu chart lớn nhất: **artifacthub.io** — tìm chart theo tên, xem
  README, values mặc định và sẵn lệnh cài.
- Cách đọc lệnh Helm install (ví dụ kubernetes-dashboard):

```bash
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard
```

Cùng một từ xuất hiện 3 lần với 3 vai trò: **release name** (tên instance
mình đặt) / **chart name** / **repository name**.

*(Transcript ghi "the help chart" — chuẩn hóa thành Helm chart; lệnh
completion tác giả gõ thiếu dấu `)` rồi tự sửa — chuẩn hóa thành `source <(helm
completion bash)`.)*

## Phần 3 — Làm việc với Helm repositories và search

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo list
```

- **Bitnami** là nhà xuất bản chart bên thứ ba (không thuộc Kubernetes) —
  chart của họ cover rất nhiều ứng dụng phổ biến.
- `helm search repo <chuỗi>`: tìm trong các repo **đã thêm vào client**, khớp
  chuỗi trong **tên hoặc mô tả** chart. Lưu ý: tìm chữ "bitnami" không có
  nghĩa là "tìm bên trong repo bitnami" — nó match mọi chart có "bitnami"
  trong tên/mô tả.

```bash
helm search repo file                 # mọi chart khớp chữ "file"
helm search repo nginx                # mọi chart khớp "nginx"
helm search repo nginx --versions     # liệt kê cả phiên bản cũ (có s)
```

- `helm repo update` — kéo index mới nhất từ remote về client; nên chạy
  trước khi install/upgrade để thấy chart bản mới.

## Phần 4 — Cài đặt ứng dụng bằng Helm — values và upgrades

### Install, status, delete

```bash
helm install bitnami/mysql --generate-name
kubectl get all                  # StatefulSet + pod mysql đầu tiên
```

- `helm install <tên> <chart>` dùng **tham số mặc định** của chart; một chart
  cài được nhiều lần — phân biệt nhau bằng tên, nên tên quan trọng.
- `--generate-name`: để Helm tự đặt tên khi không muốn tự nghĩ tên.
- `helm list` — danh sách release từ **góc nhìn Helm**; `kubectl get all` —
  từ góc nhìn Kubernetes: cùng một hệ thống, hai cách nhìn.
- `helm status <tên>` — trạng thái release: deployed, các service đã tạo,
  username và cách truy cập ứng dụng (vd mật khẩu mysql nằm trong Secret).
- `helm show chart bitnami/mysql` — mô tả chart; `helm show all` — đầy đủ cả
  values mặc định.
- `helm delete <tên>` — gỡ release.

### Values — tùy biến chart

- Chart = **templates + values**: values mặc định nằm trong `values.yaml` của
  chart; mình override bằng file values riêng, **merge** vào mặc định — chỉ
  cần khai báo phần muốn đổi.

```bash
helm show values bitnami/nginx              # rất dài
helm show values bitnami/nginx | grep replicaCount   # lấy đúng dòng cần
```

Tạo file `values.yaml` riêng:

```yaml
commonLabels:
  type: helmapp
replicaCount: 3
```

```bash
helm install nginx bitnami/nginx --values values.yaml
kubectl get deploy --show-labels    # 3 replicas + label type=helmapp
```

> **Ghi nhớ thi CKAD**: `helm upgrade` **không tự nhớ** `--values` — mỗi lần
> upgrade phải truyền lại `--values values.yaml`, không thì giá trị tùy
> chỉnh mất về mặc định.

- `--set key=value` — set từng cặp trực tiếp trên dòng lệnh; chỉ dùng khi cần
  gấp, vì values.yaml là declarative và đưa lên Git chuẩn hóa được.

### Kiểm tra và upgrade

```bash
helm get values nginx          # values mình đã override
helm get values nginx --all    # cả defaults + override
```

- Ba khái niệm cần phân biệt:
  - **Chart** — gói lấy từ repository.
  - **Installation** — instance đã cài vào cluster.
  - **Release** — chart + installation; mọi release đều upgrade được.
- Upgrade để thay đổi tham số:

```bash
helm install nginx bitnami/nginx --set ingress.enabled=false   # test nội bộ
helm upgrade nginx bitnami/nginx --set ingress.enabled=true
```

- Upgrade lên phiên bản chart mới hơn: `helm repo update` trước, rồi
  `helm upgrade nginx bitnami/nginx` — nhảy lên bản mới nhất nếu có.

*(Transcript ghi "Helm update" — chuẩn hóa thành `helm upgrade`; "replica
count" / "common labels" / "type Helm app" — chuẩn hóa thành `replicaCount`,
`commonLabels`, `type: helmapp`; "grab" → `grep`; "bit.nginx" — tách thành
release `nginx` + chart `bitnami/nginx`.)*

## Phần 5 — Kustomize

- **Kustomize** là **tính năng sẵn có của Kubernetes** — không cài thêm gì;
  hoạt động qua file `kustomization.yaml` mô tả các **thay đổi** áp lên một
  tập resources.
- Use case đặc trữ: file input **mình không kiểm soát nội dung** — vd YAML
  lấy từ Git của vendor, nội dung đổi theo từng version — không nên sửa tay.
  Kustomize giữ file gốc nguyên vẹn, tách thông tin đặc thù site ra riêng.
- Áp/xóa nguyên bộ bằng `-k`:

```bash
kubectl apply -k .     # "." = thư mục hiện tại chứa kustomization.yaml
kubectl delete -k .    # xóa mọi thứ -k đã tạo
```

### kustomization.yaml

```yaml
resources:
  - deployment.yaml
  - service.yaml
namePrefix: test-
```

- `resources` — các YAML gốc, chạy y nguyên như mặc định.
- Các option phổ biến: `namePrefix` (tiền tố tên mọi resource), `namespace`,
  `commonLabels`.

### Base và overlays

```text
base/          # config chung của ứng dụng
overlays/
  dev/         # kustomization.yaml trỏ ../../base + thay đổi riêng dev
  staging/
  prod/
```

- Mỗi overlay có `kustomization.yaml` riêng, tham chiếu base trong
  `resources` (`../../base`) cộng thêm thay đổi đặc thù môi trường — nền của
  DevOps pipeline: cùng một base, ba môi trường ba bộ overlay.

### Demo

Thư mục `kustomize` trong course repo: deployment + service +
kustomization.yaml (resources + namePrefix):

```bash
cd kustomize
kubectl apply -k .
kubectl get all              # tên resource thêm prefix test-
kubectl delete -k .
```

*(Transcript ghi "Customize" và "customization.yaml" — chuẩn hóa thành
Kustomize và `kustomization.yaml`.)*

## Phần 6 — End-of-lesson lab

### Đề bài

Cài **HashiCorp Vault** vào cluster bằng Helm, theo đúng hướng dẫn từ
artifacthub.io.

### Lời giải

1. Mở **artifacthub.io** → tìm "Vault" → chọn **HashiCorp Vault**.
2. Bấm **Install** — trang hiện sẵn đúng **hai lệnh**:

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install my-vault hashicorp/vault
```

- Tên mặc định `my-vault` — theo tác giả "không đẹp lắm nhưng chạy được";
  muốn đẹp hơn thì tự đặt tên khác.
3. Output sau install tương tự `helm status` — trạng thái release + hướng
   dẫn bước tiếp theo.

Verify:

```bash
helm get manifest my-vault    # mọi component Helm đã sinh ra
kubectl get all               # StatefulSet my-vault + pods + services
```

*(Transcript ghi "helm-get on the manifest" — chuẩn hóa thành `helm get
manifest`.)*

## Tổng hợp lệnh Helm & Kustomize

| Lệnh | Chức năng |
|---|---|
| `helm repo add <tên> <url>` | Thêm Helm repository vào client |
| `helm repo list` / `helm repo update` | Danh sách repo / kéo index mới nhất |
| `helm search repo <chuỗi> [--versions]` | Tìm chart trong repo đã thêm |
| `helm install <release> <chart> [--values f.yaml \| --set k=v]` | Cài chart |
| `helm install <chart> --generate-name` | Cài với tên tự sinh |
| `helm list` / `helm status <release>` | Danh sách release / trạng thái |
| `helm show chart \| values \| all <chart>` | Xem mô tả / values mặc định |
| `helm upgrade <release> <chart> [--values f.yaml]` | Nâng cấp release |
| `helm get values <release> [--all]` | Values đang áp dụng |
| `helm delete <release>` | Gỡ release |
| `kubectl apply -k <thư-mục>` / `kubectl delete -k <thư-mục>` | Áp/xóa qua Kustomize |

> Hết Module 3 — đã đủ công cụ chạy **và cài** ứng dụng. Module 4 chuyển sang
> **Services & Networking**: exposing applications.
