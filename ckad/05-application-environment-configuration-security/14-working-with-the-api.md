# Bài 14 — Working With the API

Bài thứ ba của Module 5 — soi vào **Kubernetes API**, thành phần mà mọi lệnh
kubectl đều đang nói chuyện với: API là gì và vận hành ra sao, dùng cURL làm
việc trực tiếp với API objects, đối phó API deprecations, và mở rộng API
bằng CustomResourceDefinitions (CRD) cùng operators.

## Lộ trình bài học

1. Understanding the Kubernetes API
2. Using cURL to work with API objects
3. Working with API deprecations
4. Understanding API extension
5. Using CustomResourceDefinitions
6. Working with operators
7. End-of-lesson lab

## Phần 1 — Understanding the Kubernetes API

- **Kubernetes API** là component cung cấp cách tương tác với Kubernetes —
  core component của mọi Kubernetes services; cung cấp **RESTful
  endpoints** cho users thao tác trên cluster.
- API **dùng và định nghĩa resources** đại diện cho các thành phần của
  cluster — pod, deployment và mọi resource đã gặp trong course.
- Hỗ trợ **declarative configuration model**: user khai báo **desired
  state** của cluster trong manifest — chủ yếu **YAML**, JSON cũng là option
  (dù ít người dùng).
- API cho phép **authentication/authorization** bằng nhiều giải pháp, và là
  **extensible** — thêm resource mới vào môi trường Kubernetes được.
- Tài liệu chính trên kubernetes.io/docs — link gắn kèm **version** (video
  dùng v1.30); đứng trên bản Kubernetes mới hơn thì thay số version, xem
  mình đang ở đâu bằng `kubectl version`. Mỗi **API group** có version
  number riêng của nó.

Hai lệnh tra cứu:

- `kubectl api-resources` — bảng resource: name, shortname, **API group**
  (api-version), cột namespaced. Nhóm đầu cho tới services thuộc **v1** —
  core Kubernetes API; nhóm đáng chú ý **apps/v1** — extension lớn đầu
  tiên của API, nơi deployment được định nghĩa.
  - **Namespaced**: resource tạo trong một namespace cụ thể và bị giới hạn
    trong biên namespace đó.
  - **Non-namespaced** (chính namespace, nodes, persistent volumes): truy
    cập từ toàn cluster.
  - Giảng viên dùng lệnh này để **verify notation** của resource và API
    version hiện hành.
- `kubectl api-versions` — chi tiết hơn về versions. Hiện API khá ổn định,
  nhưng một số resource tồn tại ở nhiều version — version mới ra thì
  version cũ thường vẫn còn, tự chọn dùng.

### Kiến trúc client — server (hình vẽ giảng viên)

- Phía cluster: **kube-apiserver** — một core Kubernetes process, chạy như
  **pod trong namespace kube-system**, cung cấp truy cập API.
- **kubectl** — utility chính giao tiếp với API; dùng file **.kube/config**
  chứa TLS keys cho giao tiếp an toàn.
- **`kubectl proxy`** — chạy proxy trên client machine; proxy đọc config và
  thiết lập kết nối **TLS-secured** tới API server. Muốn dùng curl mà không
  lo TLS certificates: curl gửi request **không bảo mật tới proxy trên
  localhost** (vô hại vì nằm trong máy mình), proxy đảm bảo đoạn truyền ra
  mạng được **TLS encrypted**.

*(Transcript ghi "kube-proxy" — lệnh thật là `kubectl proxy`; kube-proxy là
network component khác của Kubernetes.)*

### Cấu hình client — kubectl config

- `kubectl config view` — xem cấu hình client hiện hành: nội dung file
  `.kube/config` với ba thành phần:

| Thành phần | Nội dung |
|---|---|
| **cluster** | Certificates + API endpoint để contact kube-apiserver |
| **users** | TLS certificates tạo nên user account |
| **context** | Tổ hợp user + cluster, kèm default namespace |

### Demo — kubectl config set-context

```bash
kubectl config set-context --help        # tra cách dùng (tab completion hỗ trợ)
kubectl config set-context --current --namespace=kube-system
kubectl get pods                         # giờ là pods của kube-system
kubectl config set-context --current --namespace=default
kubectl get pods                         # về lại default
```

- `--current` sửa **context hiện tại** (minikube): default namespace đổi
  sang kube-system, mọi lệnh sau đó chạy trong namespace đó.
- Đây là **cách duy nhất kubectl mặc định cung cấp để đổi current
  namespace** — tiện khi muốn "đứng" hẳn trong một namespace; nhưng phải nhớ
  quay về điểm xuất phát, không sẽ tạo resource sai namespace.

> **Ghi nhớ thi CKAD**: `kubectl config set-context --current
> --namespace=<ns>` đổi default namespace của context — dùng xong set về
> `default`.

## Phần 2 — Using cURL to work with API objects

**Không nằm trong đề CKAD** — giảng viên nói rõ đây không phải key skill,
không cần memorize cho exam; giá trị là insight hiểu Kubernetes vận hành.

- Mở proxy trên workstation: `kubectl proxy --port=8001 &` — nhớ `&`, không
  là nó chiếm foreground.
- `curl http://localhost:8001` — thấy mọi **API paths và groups** khả dụng,
  cửa vào tất cả các exposed functions.

### Demo — curl các API objects

```bash
kubectl proxy --port=8001 &                     # Starting to serve on 8001
kubectl run proxypod --image=nginx              # pod dummy cho demo
curl http://localhost:8001/version
curl http://localhost:8001/api/v1/namespaces/default
curl http://localhost:8001/api/v1/namespaces/default/pods/proxypod
curl -X DELETE http://localhost:8001/api/v1/namespaces/default/pods/proxypod
```

- `/version` — proxy forward thẳng tới kube-apiserver; curl không cần lo
  TLS vì proxy lo phần mã hóa.
- `/api/v1/namespaces/default` — mọi thứ trong namespace default; nối thêm
  `/pods/proxypod` — đầy đủ properties của pod đó.
- `curl -X DELETE` tới URL của pod — xóa pod; pod lỗi CrashLoopBackOff
  cũng bị dọn sạch luôn.
- Bẫy trong demo (giữ nguyên làm bài học): gõ nhầm `kubectl run pod
  proxypod` — `kubectl create` cần resource type, còn **`kubectl run`
  không cần**; kết quả tạo ra pod tên `pod` chạy image `proxypod` không tồn
  tại → crash loop backoff. `kubectl get pods` lộ ra ngay.
- Vì sao dùng cách này: ứng dụng không có Kubernetes client hoàn chỉnh vẫn
  interface được với Kubernetes qua REST qua proxy.

## Phần 3 — Working with API deprecations

- API versions thay đổi — Kubernetes có bản mới **cứ 4 tháng**, old API
  versions có thể bị **deprecated**.
- Chính sách chính thức: version bị deprecated còn được hỗ trợ **tối thiểu
  2 Kubernetes releases nữa**, rồi bị gỡ hẳn. Thấy deprecation message →
  phải hành động, sửa YAML manifests:
  - Đơn giản: chỉ đổi `apiVersion` trong YAML.
  - Phức tạp: cả cách định nghĩa resource đổi — regenerate YAML theo spec
    mới.

### Demo — sửa redis-deploy.yaml bị deprecated

```bash
kubectl create -f redis-deploy.yaml
# error: no matches for kind Deployment in version apps/v1beta1
kubectl api-versions                    # hiện chỉ còn apps/v1
kubectl explain deploy --recursive | less
```

- File trong course Git repo dùng `apiVersion: apps/v1beta1` — version đã
  bị gỡ khỏi cluster nên `create` fail ngay với lỗi resource mapping not
  found.
- `kubectl explain --recursive` — giải thích object cùng **toàn bộ child
  objects và cháu của chúng**; output cực dài nên thu hẹp thành
  `kubectl explain deploy.spec --recursive`, đối chiếu các property đang
  dùng xem cái nào cần cập nhật.
- Sửa `apps/v1beta1` → `apps/v1` rồi apply. Chiến thuật của giảng viên:
  **không soi trước** — quá tốn công; apply rồi xem Kubernetes kêu gì. Lần
  này may mắn chạy ngay; khi gặp complaint về **unknown attributes** thì
  mới lôi documentation ra xử lý.

*(Transcript ghi "redis deploy.yaml" — chuẩn hóa thành redis-deploy.yaml.)*

## Phần 4 — Understanding API extension

Ba cách mở rộng Kubernetes API:

| Cách | Bản chất | Nhận xét |
|---|---|---|
| **Custom Resource Definition (CRD)** | API resource giúp thêm resource riêng vào API một cách dễ dàng | Tích hợp thẳng API server, theo update mechanism của nó; **không cần lập trình** — de facto standard những bản gần đây |
| **Custom controller** | Process **watch** thay đổi trong Kubernetes API, hành động để giữ desired state của resource | Tự động hóa tác vụ trong cluster; nói chuyện với API qua **client libraries**; thường phân phối sẵn qua operators |
| **API aggregation** | Bổ sung **additional API servers** | Customization cấp cao, tích hợp resources từ nhiều nguồn — nhưng API server phải tự lập trình: công sức lớn, skill riêng |

- Phổ biến nhất: **CRD** — chi tiết ở Phần 5.

## Phần 5 — Using CustomResourceDefinitions

- **CRD** cho phép thêm custom resources vào cluster — tích hợp bất cứ thứ
  gì vào môi trường cloud-native. Resource mới là **extension của chính
  API server gốc**, hiện ra trong `kubectl api-resources`; điểm hay:
  **không cần kỹ năng lập trình**.
- Điều duy nhất cần nhớ: custom resource chỉ có nghĩa khi có **application
  dùng nó** — không có pod tương ứng đọc resource đó mà làm việc trong
  cluster thì thêm chỉ để ngắm.
- Quy trình: **định nghĩa resource bằng kind CustomResourceDefinition**
  trước; sau đó nó trở thành API resource riêng, xài như resource thường.

### Demo — CRD backups

File `crdobject.yaml` (course Git repo) định nghĩa resource
`backups.stable.example.com`:

- **Name** phải là **fully qualified name**: `backups` + API group
  `stable.example.com`.
- `versions` — hiện v1.
- **Schema viết bằng OpenAPI v3**: type object với ba properties —
  `backupType`, `image` (string) và `replicas` (integer) — giữ đơn giản cho
  demo.
- `scope: Namespaced` — resource này namespaced (CRD khai báo rõ namespaced
  hay không), kèm các **naming attributes** bắt buộc.

```bash
kubectl apply -f crdobject.yaml
kubectl api-resources | grep backup       # resource backups đã vào API
```

Dùng resource mới — file `crdbackup.yaml`: đúng apiVersion, `kind: Backup`,
tên trong `metadata`, `spec` gán giá trị cho từng property:

```bash
kubectl apply -f crdbackup.yaml
kubectl get backups                       # custom resource của ta nằm trong API
```

- Hữu ích chừng nào? Tự thân — **không chút nào**; demo chỉ để hiểu cơ chế
  vận hành. Real life: custom resource cần **application đi kèm** tiêu thụ
  nó — như ingress controller *(Bài 11)* chính là application đọc resources
  từ API để làm việc.

## Phần 6 — Working with operators

- **Operators** = custom applications xây trên **CRD** — cách
  **packaging, running và managing** applications trong Kubernetes.
- Operators đứng trên **controllers** — Kubernetes components liên tục vận
  hành dynamic systems. Cốt lõi là **controller loop**: controller manager
  chạy **reconciliation loop** — quan sát **current state**, so với
  **desired state**, điều chỉnh khi cần thiết. Operators =
  **application-specific controllers**.
- Nguồn operators: tự phát triển, hoặc lấy từ community — registry quen
  thuộc: **operatorhub.io** (hơi thiên hướng Red Hat OpenShift nhưng nhiều
  thứ tốt).
- Nhiều giải pháp của hệ sinh thái Kubernetes phát hành dạng operator:
  **Prometheus** (monitoring & alerting), **Tigera** (operator quản lý
  Calico network plugin), **Jaeger** (tracing transactions giữa các
  distributed systems).
- Demo OperatorHub: tìm **NFS** → NFS operator và NFS provisioner operator
  — cần storage thật qua NFS provisioner thì đây là điểm khởi đầu; nút
  Install chỉ dẫn từng bước. Cài operator thường nhận được **CRD +
  application đi kèm** — giải pháp hoàn chỉnh tích hợp thẳng vào
  Kubernetes; operators đang trở thành cách phổ biến để tích hợp
  third-party applications.

*(Transcript ghi "Tygera" — chuẩn hóa thành Tigera.)*

## Phần 7 — End-of-lesson lab — Working With the API

### Đề bài

Tìm mọi resource trên hệ thống đã được định nghĩa bằng custom resource
definitions.

### Lời giải

```bash
kubectl api-resources | grep -i custom
kubectl get crds
```

- `grep -i custom` lộ resource **customresourcedefinitions** kèm shortname
  **crd / crds** — chính là API resource dùng để tạo CRD.
- `kubectl get crds` liệt kê **mọi CRD hiện có** trong cluster — toàn bộ
  những gì đã được định nghĩa bằng CRD. *"Now that was easy, wasn't it?"*

## Tổng hợp lệnh Bài 14

| Lệnh | Chức năng |
|---|---|
| `kubectl api-resources` / `kubectl api-versions` | Tra resource, API group và version hiện hành của cluster |
| `kubectl config view` | Xem cấu hình client trong `.kube/config`: cluster, users, context |
| `kubectl config set-context --current --namespace=<ns>` | Đổi default namespace của context hiện tại — xong nhớ set về `default` |
| `kubectl proxy --port=8001 &` + `curl http://localhost:8001/...` | Làm việc trực tiếp với API bằng curl — proxy giữ phần TLS |
| `kubectl explain <resource> --recursive` | Soi toàn bộ child properties — đối chiếu YAML khi gặp deprecation |
| `kubectl get crds` | Liệt kê mọi custom resource definitions trong cluster |

> Từ ConfigMaps, Secrets qua DevOps deployments tới chính Kubernetes API,
> ứng dụng giờ đã được cấu hình, triển khai và mở rộng theo chiều sâu của
> platform. Bài 15 — **Security** — bài cuối của Module 5, khép lại miền
> Application Environment, Configuration and Security bằng các tùy chọn
> security cho workload trong cluster.
