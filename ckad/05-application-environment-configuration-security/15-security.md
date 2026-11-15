# Bài 15 — Security

Bài cuối của Module 5 — các tùy chọn **security** cho workload trong cluster:
authentication và authorization diễn ra ra sao khi kết nối tới Kubernetes
cluster, pod dùng **service accounts** truy cập các Kubernetes resources
khác, **role-based access control** (RBAC) gắn quyền cho service accounts,
cùng **security context** và **resource requests/limits** với quota.

## Lộ trình bài học

1. Understanding authentication and authorization
2. Using service accounts
3. Understanding role-based access control
4. Working with security context
5. Working with resource requests, limits, and quota
6. End-of-lesson lab

## Phần 1 — Understanding authentication and authorization

- **Authentication** — user đến từ đâu. Trong vanilla Kubernetes và Minikube,
  một **local Kubernetes admin account** đảm nhiệm authentication. Setup nâng
  cao hơn có thể tạo user accounts riêng — phạm vi lớp CKA.
- `.kube/config` chỉ định cluster mà users authenticate tới; xem bằng
  `kubectl config view` hoặc đọc thẳng file *(nhắc lại Bài 14 Phần 1)*.
- **Authorization** — user đó được làm gì. Đứng sau là **role-based access
  control (RBAC)** lo các tùy chọn này.
- Tra quyền hiện có: `kubectl auth can-i <verb> <resource>` — ví dụ
  `kubectl auth can-i get pods`.

### Demo — kubectl auth can-i

```bash
kubectl auth can-i get pods
kubectl auth can-i get pods --as=bob@example.com
```

- Lệnh đầu: **yes** — admin hiện hành được get pods.
- Lệnh sau: `bob@example.com` là user **không tồn tại** → không có
  authorization nào diễn ra → **no**.
- `--as` test quyền của danh tính khác, kể cả service account:

```bash
kubectl auth can-i get pods --as=system:serviceaccount:bellview:viewer -n bellview
```

  "Với tư cách service account viewer trong namespace bellview, tôi có được
  get pods không?" — cú pháp `system:serviceaccount:...` sẽ rõ ở Phần 3.

## Phần 2 — Using service accounts

- Mọi action trong Kubernetes cluster đều phải được **authenticated và
  authorized**.
- **Service accounts** = cách authentication cơ bản **từ bên trong** cluster.
- **RBAC** nối service account vào một role cụ thể.
- Mọi pod đều dùng **default service account** để kết nối tới API server.
  Default SA chỉ cho phép resource **lấy thông tin từ API server** — không
  hơn.
- Mỗi service account dùng một **secret** để tự động hóa API credentials
  trong application.

*(bổ sung từ nghiên cứu ngoài video)* Từ Kubernetes 1.24, token không còn tự
sinh thành Secret cho mỗi SA; pod nhận token tạm (tự rotate) qua projected
volume do TokenRequest API cấp.

- Đa số pod dùng default SA là đủ. Pod cần **truy cập resources trong
  cluster** → cần **custom SA** nối tới role cụ thể qua **role binding** —
  trường hợp của network plugins, monitoring software và các components
  bổ sung cài vào Kubernetes.

### Demo — service accounts hiện có

```bash
kubectl describe pod redis-<suffix>     # pod nào cũng được
kubectl get sa -n kube-system
kubectl get sa
kubectl describe pod coredns-<suffix> -n kube-system
```

- `describe pod` — không xa đầu output: `Service Account: default`. Mỗi
  namespace có sẵn một SA tên **default**.
- `kubectl get sa -n kube-system` — cả đống: kube-system là nơi ở của system
  components, và chúng cần truy cập nhiều hơn.
- Ở default namespace chỉ thấy SA default — ứng dụng người dùng thường
  không cần SA riêng.
- Pod CoreDNS dùng SA **coredns** — CoreDNS cần communicate với pods để
  cung cấp DNS credentials.

## Phần 3 — Understanding role-based access control

- RBAC dùng **ba thành phần** để cấp permission cho API objects:

| Thành phần | Vai trò |
|---|---|
| **Role** | Tập **verbs** gán permission cụ thể — get, list, watch... |
| **Service account** | Danh tính pods dùng để truy cập API resources |
| **Role binding** | Nối service account tới role |

- Role và Role binding có **namespace scope**. **ClusterRole** và
  **ClusterRoleBinding** làm đúng việc đó nhưng **cluster scope** — áp dụng
  cho toàn cluster.
- Ngoài service accounts, RBAC còn có **users** — cho con người cần truy
  cập resources cụ thể. **Không nằm trong đề CKAD** (thuộc CKA).

### Demo — dựng chuỗi RBAC trong namespace bellview

```bash
kubectl create ns bellview
kubectl create role viewer --verb=get --verb=list --verb=watch \
  --resource=pods -n bellview
kubectl create sa viewer -n bellview
kubectl create rolebinding viewer --serviceaccount=bellview:viewer \
  --role=viewer -n bellview
```

- Không nhớ cú pháp? `kubectl create role --help` có sẵn examples khá tốt —
  chính giảng viên cũng dựa vào đó mà viết.
- Trong `create rolebinding`, service account viết dạng
  **namespace:resource name** (`bellview:viewer`) — vì có thể lấy SA từ
  namespace khác gắn vào role của namespace này.

Gắn SA vào deployment:

```bash
kubectl create deploy viewginx --image=nginx --replicas=3 -n bellview
kubectl set serviceaccount deploy/viewginx viewer -n bellview
```

- `kubectl set serviceaccount` cập nhật SA cho deployment — từ giờ
  deployment chạy với permissions tương ứng.

Verify bằng auth can-i:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:bellview:viewer -n bellview
kubectl auth can-i get pods \
  --as=system:serviceaccount:bellview:viewer -n default
```

- Test với tư cách SA luôn bắt đầu `system:serviceaccount:`, tiếp theo là
  **namespace**, tiếp là **tên SA**.
- Kết quả: trong bellview → **yes**; trong default → **no** — role viewer
  có namespace scope nên đúng thiết kế.

### Demo — ClusterRoleBinding của CoreDNS

```bash
kubectl describe sa coredns -n kube-system
kubectl describe clusterrolebinding system:coredns
kubectl describe clusterrole system:coredns
```

- `describe sa` cho thấy bản thân service account là resource gần như
  trống — muốn biết nó làm gì phải xem (cluster) role binding gắn với nó.
- `system:coredns`: RoleRef kind **ClusterRole**, Subjects là SA
  **coredns** — CoreDNS là hệ thống then chốt của Kubernetes, cần access
  **toàn cluster**, nên dùng ClusterRoleBinding thay RoleBinding.
- `describe clusterrole system:coredns` — liệt kê **verbs** trong
  **PolicyRules** của role. Đó là nơi service accounts, roles và role
  bindings gặp nhau.

> **Ghi nhớ thi CKAD**: Đề **không đòi tạo roles/rolebindings** — kỳ vọng là
> **set đúng service account cho resource của mình**. Ứng dụng fail →
> `kubectl describe pod` bắt thông báo liên quan RBAC → `kubectl get sa -n
> <ns>` xem có SA khác sẵn có → `kubectl set serviceaccount deploy/<tên>
> <sa> -n <ns>` (tra cú pháp bằng `-h`). Nắm được mức đó là đủ cho exam.

## Phần 4 — Working with security context

- **Security context** định nghĩa **privilege và access control settings**
  cho pod hoặc container. Các option tiêu biểu:

| Option | Ý nghĩa |
|---|---|
| `allowPrivilegeEscalation` | Container có được chạy với root privileges không |
| `capabilities` | POSIX capabilities mà container được dùng |
| `runAsNonRoot` | Ép chạy với user ID non-privileged |
| `readOnlyRootFilesystem` | Không cho ghi vào file system của container |
| `runAsUser` | Chạy container với một user cụ thể |
| `seLinuxOptions` | SELinux context labels |

- Đây chưa phải danh sách đầy đủ — xem hết bằng
  `kubectl explain pod.spec.securityContext` (mức pod) và
  `kubectl explain pod.spec.containers.securityContext` (mức container).
- Security context áp dụng cho **pod cũng như container** — hợp lý trong
  multi-container pod: cho phép container này làm điều mà container kia
  không được.
- Security context ngăn pod chạy thành công → `kubectl describe` lấy thêm
  thông tin từ events. Pod fail vì security context có thể hiện là failing
  **hoặc pending**.

### Demo — security context ở cả hai mức

File `security-context-demo-2.yaml` có securityContext trong **pod spec**
lẫn **container spec**:

- Settings mức pod **không nhất thiết trùng** mức container — bối rối thì
  `kubectl explain` cả hai.
- Mức pod: `runAsUser: 1000`, `runAsGroup: 3000`, `fsGroup: 2000` — định
  nghĩa user và group cho containers của pod.
- Mức container: `allowPrivilegeEscalation: false` — không thể sudo hay gì
  tương tự từ trong container.

```bash
kubectl exec -it security-context-demo -- sh
ps                                   # processes chạy với user ID 1000
cd /data && ls -l                    # demo: owner root, group 2000
cd demo && echo hello > testfile && ls -l
```

- `ps`: mọi process khởi động bởi **user ID 1000** — đúng runAsUser.
- `/data/demo` owned bởi root, group **2000** (fsGroup).
- File mới tạo owned bởi user ID 1000 và group 2000 — **không phải**
  primary group 3000 (runAsGroup): **fsGroup** đặt group mặc định cho file
  mới tạo từ trong container.

### Demo — runAsNonRoot đụng image chạy root

File `security-context-demo.yaml`: pod `nginx-secure` với
`runAsNonRoot: true` — cố chạy container non-root:

```bash
kubectl apply -f security-context-demo.yaml
kubectl get pods                     # CreateContainerConfigError
kubectl describe pod nginx-secure
```

- Event cho thấy xung đột: container có **runAsNonRoot** nhưng **image chạy
  root** — mâu thuẫn nên pod không thể chạy.

## Phần 5 — Working with resource requests, limits, and quota

- **Resource request** — setting trên container
  (`pod.spec.containers.resources.requests`): yêu cầu **tối thiểu**;
  container chỉ được schedule lên node đáp ứng đủ request.
- **Resource limit** (`pod.spec.containers.resources.limits`): mức **tối
  đa** được phép cấp phát — giữ ứng dụng không vượt quá ngưỡng nhất định.
- Cả hai đều là **container property**.
- Đơn vị: memory và CPU. CPU tính bằng **millicores (millicpu)** — 1/1000
  CPU core: 500m = 0.5 CPU.
- Khi schedule, **kube-scheduler** bảo đảm node chạy pod có đủ requested
  resources. Pod có resource settings mà không schedule được → trạng thái
  **pending**.
- Bẫy dễ vấp: node 1 CPU, deployment 3 replicas × 500m = 1.5 CPU — node
  không bao giờ chứa đủ 3 replicas này; pods chiếm hết chỗ thì node không
  còn available cho ai nữa.
- `kubectl set resources` — set requests/limits trên deployment đang chạy.

> **Ghi nhớ thi CKAD**: đề yêu cầu set requests/limits trên pod riêng lẻ →
> cân nhắc tạo deployment, `kubectl set resources` lên deployment, rồi
> copy-paste cấu hình từ deployment YAML sang pod YAML — mẹo nhanh cho
> exam.

### Quota — không thuộc đề CKAD

**Không nằm trong đề CKAD** — trình bày để có bức tranh đầy đủ.

- **Quota** = restrictions đặt trên **namespace** để tối đa hóa
  availability của resources trong namespace đó; cũng set được limits ở
  mức quota.
- Namespace có quota → mọi pod chạy trong đó **bắt buộc** phải có resource
  requests và limits.
- `kubectl create quota -n <ns>` để áp dụng.

### Demo — frontend-resources.yaml

File có một pod với 2 containers:

- **mysql** (kèm environment variable): request **64Mi memory / 250m
  CPU**, limit **128Mi / 500m**.
- **wordpress**: cũng có request và limit tương tự — cách làm chuẩn.

```bash
kubectl apply -f frontend-resources.yaml
kubectl get pods
kubectl describe pod frontend
```

- Container WP chạy; container DB bị **OOMKilled** — OOM = Linux out of
  memory: process bị hạ vì không lấy được memory nó yêu cầu.
- `describe`: phần generic chỉ nói restarted failing container — cuộn lên
  thấy WP ready còn DB container **terminated OOMKilled**: limit 128MB
  không đủ cho database.

Sửa theo ba lượt:

1. `kubectl delete -f frontend-resources.yaml`; edit request thành **64Gi**
   (phi lý — cố tình để minh họa), tăng cả limits → apply → pod **pending**.
   `describe` cho thấy **FailedScheduling — insufficient memory** — pending
   là trạng thái chuẩn khi scheduler không tìm được node đáp ứng request.
2. Edit lại: request **64Mi**, limit **512Mi** — thực tế hơn cho database →
   apply → pod **running**.
3. Hoặc set thẳng trên deployment trong một lệnh duy nhất:

```bash
kubectl set resources deploy/<tên> --requests memory=64Mi,cpu=250m \
  --limits memory=512Mi,cpu=500m
```

## Phần 6 — End-of-lesson lab — Security

### Đề bài

Trong namespace `secure`: tạo service account tên `secure`, khởi động
deployment tên `secure-deploy` và bảo đảm nó dùng service account
`secure`. Không cần thêm gì nữa — role-based access control không thuộc
phạm vi CKAD.

### Lời giải

```bash
kubectl create ns secure
kubectl create sa secure -n secure
kubectl create deploy secure-deploy --image=nginx -n secure
kubectl set serviceaccount deploy/secure-deploy secure -n secure
kubectl describe -n secure pod secure-<suffix>
```

- Kiểm chứng cuối: `describe` pod cho thấy pod đang dùng service account
  **secure**.

## Tổng hợp lệnh Bài 15

| Lệnh | Chức năng |
|---|---|
| `kubectl auth can-i <verb> <resource>` | Tra quyền; `--as=system:serviceaccount:<ns>:<sa>` test quyền của SA |
| `kubectl get sa -n <ns>` | Liệt kê service accounts trong namespace |
| `kubectl set serviceaccount deploy/<tên> <sa> -n <ns>` | Gán SA cho deployment — thao tác RBAC duy nhất đề CKAD đòi |
| `kubectl explain pod.spec.containers.securityContext` | Toàn bộ options của security context (pod và container level) |
| `kubectl set resources deploy/<tên> --requests ... --limits ...` | Set requests/limits trên deployment đang chạy |
| `kubectl describe pod` | Bắt sự cố: RBAC events, security context, OOMKilled, FailedScheduling |

> Module 5 khép lại: ứng dụng được cấu hình bằng ConfigMaps và Secrets,
> triển khai theo cách DevOps, mở rộng qua Kubernetes API và nay được bảo
> vệ bằng security settings. Module 6 — **Observability & Maintenance** —
> chuyển sang phía vận hành: quan sát và troubleshoot khi sự cố xảy ra.
