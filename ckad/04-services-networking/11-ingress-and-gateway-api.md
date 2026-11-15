# Bài 11 — Ingress and Gateway API

Bài cuối của Module 4 — miền **Services and Networking** (20% đề thi). Ingress
là **reverse HTTP proxy** tích hợp sâu vào Kubernetes và đang chuyển giao cho
Gateway API thế hệ mới. Bài này phủ cài Ingress controller, viết Ingress rules,
cấu hình Gateway API và troubleshooting networking khi traffic không đến pod.

## Lộ trình bài học

1. Managing incoming traffic
2. Ingress components
3. Installing ecosystem Ingress controllers
4. Using the Minikube Ingress controller
5. Using Ingress
6. Configuring Ingress rules
7. Understanding Gateway API
8. Configuring Gateway API
9. Using Gateway API to provide access to applications
10. Troubleshooting networking
11. End-of-lesson lab

## Phần 1 — Managing incoming traffic

- NodePort (Bài 10) expose được mọi port nhưng thô với HTTP. Ingress là cách
  tinh hơn — và bản thân nó đang trên đường thay thế:

| Công nghệ | Trạng thái |
|---|---|
| **Ingress** | Trong **feature freeze** — chỉ bảo trì, không thêm tính năng; **vẫn nằm trong CKAD objectives** |
| **Gateway API** | Thế hệ thay thế *(Bài 10 Phần 6)*; thời điểm quay video **chưa nằm trong CKAD objectives** |

- Đề thi định nghĩa bởi **CKAD objectives** — hai nguồn kiểm tra: trang Linux
  Foundation (domains & competencies, hiện vẫn ghi "use ingress rules") và
  GitHub repo `cncf/curriculum` — cập nhật thường xuyên, tự chịu trách nhiệm
  check PDF trước ngày thi.
- Bài này học cả hai: Ingress để thi, Gateway API để làm việc thực tế.

> **Ghi nhớ thi CKAD**: objectives thay đổi — kiểm tra `cncf/curriculum` trên
> GitHub trước khi thi. Ở thời điểm quay video, Ingress vẫn trong đề, Gateway
> API thì chưa.

- Cảnh báo thực hành *(kinh nghiệm giảng viên)*: đừng cấu hình Ingress và
  Gateway API **cùng lúc trên một máy** — "they will bite one another".

## Phần 2 — Ingress components

- **Ingress** là API resource cung cấp external access tới internal resources.
  Traffic đi qua một external load balancer, và toàn bộ cơ chế do **Ingress
  controller** — một ứng dụng chạy trong cluster — implement.
- Chia trách nhiệm rõ hai phần:

| Thành phần | Vai trò |
|---|---|
| **Ingress resource** | Khai báo rules — cấu hình routing |
| **Ingress controller** | Đọc rules và **thực thi** routing — reverse HTTP proxy chạy như pod |

- **Ingress resource mà không có controller = không có tác dụng gì cả**. Và
  Kubernetes **không cài controller mặc định** — CNCF trung lập distributor,
  không chọn sẵn một controller nào.
- Controllers phổ biến: **Nginx** (course dùng), HAProxy, Traefik, Kong,
  Contour.
- Luồng traffic:

```text
user ──► external DNS ──► Ingress (reverse HTTP proxy) ──► rule khớp ──► Service ──► pods
```

- Hostname trong rule phải **resolve** về IP của load balancer / node — qua DNS
  thật hoặc file `/etc/hosts` trên máy client (demo dùng minikube host IP).
- Rules chính là "configuration file" của reverse proxy: thay vì file config
  trên disk, controller pod đọc **Ingress resource** — quản lý bằng kubectl như
  mọi resource khác.
- Tùy controller, Ingress cung cấp: externally reachable URLs, load balancing,
  **SSL/TLS termination**, **name-based virtual hosting**.
- Giới hạn so với NodePort: Ingress **chỉ làm việc với HTTP/HTTPS**.

*(Transcript ghi "Traffic" — chuẩn hóa thành Traefik; "reversed HTTPS proxy" —
chuẩn hóa thành reverse HTTP proxy.)*

## Phần 3 — Installing ecosystem Ingress controllers

- Vanilla Kubernetes không có Ingress controller — mỗi distributor cung cấp
  cách cài riêng. Cách **distribution-independent**: Helm.
- Cài **ingress-nginx** vào namespace riêng (demo chạy trên cluster không phải
  minikube):

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
kubectl get pods -n ingress-nginx
```

- Pod lúc đầu **0/1 Ready** là bình thường — controller cần thời gian khởi
  động.

*(Transcript ghi "kubernetes.github.io/.ingress-nginx" — chuẩn hóa thành
https://kubernetes.github.io/ingress-nginx.)*

## Phần 4 — Using the Minikube Ingress controller

- Minikube có sẵn addon ingress, tắt mặc định để giữ lightweight:

```bash
minikube addons list             # ingress: disabled
minikube addons enable ingress   # pull images — mất khoảng một phút
```

### Demo — troubleshooting pod Pending

Lệnh enable treo ở "verifying ingress addon" → lần theo từng lớp:

```bash
kubectl get ns                             # ns ingress-nginx đã tồn tại
kubectl get all -n ingress-nginx           # pod controller Pending
kubectl -n ingress-nginx describe pod ingress-nginx-controller-...
# Events: Insufficient cpu ... 0/1 nodes available ... no preemption candidates
```

- **Nguyên nhân**: pod khai báo **resource requests** CPU (nhiều container ×
  0.5 core) vượt tổng CPU của node (demo: node 2 cores) → không node nào nhận.
- **Hai hướng xử lý**: giảm requests trong manifest (resources/limits — module
  sau), hoặc **thêm CPU cho node**.
- Demo tăng VM 2 → 8 cores → pod **Pending → ContainerCreating** (đã qua
  scheduling, chỉ chờ pull image) → **Running**.

*(Transcript ghi "no preemption fictions" — chuẩn hóa thành no preemption
candidates.)*

## Phần 5 — Using Ingress

- Prereq *(từ demo Bài 10)*: deployment `nginxsvc` 3 pods + Service port 80 —
  `kubectl get all -l app=nginxsvc`.
- Xem syntax đầy đủ kèm example: `kubectl create ing -h | less`.

### Demo — Ingress đầu tiên

```bash
kubectl create ingress nginxsvc --rule="/=nginxsvc:80"
sudo vim /etc/hosts          # thêm dòng: <minikube-ip> nginxsvc.info
kubectl describe ing nginxsvc
curl nginxsvc.info           # Welcome to nginx
```

- `--rule="/=nginxsvc:80"`: path `/`, không host → mọi request forward sang
  Service `nginxsvc` port 80.
- `describe ing`: Hosts `*`, backend `nginxsvc:80` kèm **pod IPs** — bằng
  chứng controller đã nối endpoints của Service.
- Luồng của `curl nginxsvc.info`: name resolution (`/etc/hosts`) → minikube
  host IP → ingress controller lắng nghe → rule khớp → Service → pod.

> **Ghi nhớ thi CKAD**: Ingress trong đề CKAD đơn giản — `kubectl create
> ingress` với `--rule` là đủ, không cần cấu hình TLS.

*(Transcript đọc cờ `--rule=` thành "rule star" — chuẩn hóa lại theo cú pháp
thật.)*

## Phần 6 — Configuring Ingress rules

### Cấu trúc một rule

| Thành phần | Bắt buộc | Ý nghĩa |
|---|---|---|
| `host` | Không | Name-based virtual host — bỏ trống thì rule áp cho mọi inbound HTTP traffic |
| `path` | Có ≥1 | Path so khớp request; mỗi path trỏ một backend |
| `backend` | Có | Service nhận traffic |

- Thường có thêm **default backend** — đón traffic không khớp rule nào.
- **pathType** — cách so khớp path:

| pathType | Hành vi |
|---|---|
| `Exact` | So khớp chính xác — `/foo` **không** match `/foo/` |
| `Prefix` | Theo tiền tố — `/` match mọi thứ; `/foo` match cả `/foo/` |

- `kubectl create ingress` sinh pathType **mặc định `Exact`** — muốn `Prefix`
  phải edit lại sau.

### Ba kiểu Ingress

1. **Single service** — một rule, một backend (demo Phần 5).
2. **Simple fan-out** — nhiều path, mỗi path một service:

```bash
kubectl create ingress single --rule="files=fileservice:80" --rule="db=dbservice:80"
```

3. **Name-based virtual hosting** — host + path, mỗi host cần DNS entry riêng
   (hoặc dòng `/etc/hosts`):

```bash
kubectl create ingress multi-host \
  --rule="myexample.com/files=fileservice:80" \
  --rule="myexample.org/data=dataservice:80"
```

### Demo — multi-host Ingress

```bash
kubectl create deployment mars --image=nginx
kubectl create deployment saturn --image=nginx
kubectl expose deployment mars --port=80
kubectl expose deployment saturn --port=80
sudo vim /etc/hosts     # mars.example.com + saturn.example.com → minikube IP
ping mars.example.com   # resolve được
kubectl create ingress multihost \
  --rule="mars.example.com/=mars:80" \
  --rule="saturn.example.com/=saturn:80"
kubectl edit ing multihost      # pathType: Exact → Prefix
```

- Path **bắt buộc** trong cú pháp rule nên host phải kèm `/` đuôi
  (`mars.example.com/`); host mới là phần optional.
- Test: `curl mars.example.com` và `curl saturn.example.com` đều trả trang
  default của nginx; `curl bogus.example.com` fail **ngay ở name resolution** —
  host chưa được khai báo ở đâu.

## Phần 7 — Understanding Gateway API

- Ingress feature freeze (Phần 1) → **Gateway API** là thế hệ kế tiếp. Thời
  điểm quay video **chưa nằm trong CKAD objectives** — phần này học để hiểu,
  và check lại objectives như Phần 1 trước khi thi.
- Gateway API cung cấp các resource qua **CRD — Custom Resource Definition** —
  cơ chế thêm object type riêng vào Kubernetes *(nhìn trước — chi tiết ở
  Bài 14)*.
- Ba resource chính:

| Resource | Ý nghĩa |
|---|---|
| **GatewayClass** | Đại diện một gateway controller |
| **Gateway** | Instance hạ tầng xử lý traffic |
| **HTTPRoute** | Định nghĩa request được route tới service nào |

- Thuật ngữ hay lẫn lộn: tên chính thức là **GatewayClass**, nhưng người ta
  thường gọi thẳng "gateway controller".
- Cần một **Gateway API controller** cài trong cluster — vai trò như Ingress
  controller với Ingress. Course dùng **Nginx Gateway Fabric** (cài bằng Helm).
- Thứ tự bắt buộc: cài **custom resources (CRD) trước, controller sau** —
  controller đọc những resource này ngay khi khởi động.

## Phần 8 — Configuring Gateway API

Đi qua từng resource:

- **GatewayClass**: `spec.controllerName` trỏ controller cụ thể. GatewayClass
  **không chứa cấu hình gì thêm** — cấu hình thật nằm ở Gateway.
- **Gateway**: `spec.gatewayClassName` nối về GatewayClass — nhiều Gateway dùng
  chung một controller được. Gateway định nghĩa **listeners** — protocol và
  port lắng nghe traffic.
- **HTTPRoute**: `spec.hostNames` nhận diện request vào; `parentRefs` nối
  HTTPRoute với Gateway; `backendRefs` trỏ Service nhận traffic.

Luồng đầy đủ:

```text
user ──► DNS ──► Service (expose controller) ──► GatewayClass ──► Gateway ──► HTTPRoute ──► Service ──► pods
```

- Theo giảng viên: "whatever you do, the Kubernetes service is still the most
  important thing" — Service vẫn là mắt xích trung tâm nối controller với pods.

## Phần 9 — Using Gateway API to provide access to applications

Quy trình 6 bước:

1. Đảm bảo custom resources (CRD) có trong cluster.
2. Cài community controller (Nginx Gateway Fabric).
3. Verify controller Ready.
4. Tạo application + Service.
5. Cấu hình Gateway + HTTPRoute.
6. Test qua Service expose controller.

### Demo phần 1 — cài CRD và controller

```bash
kubectl apply -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
helm install ngf oci://ghcr.io/nginx-inc/charts/nginx-gateway-fabric \
  --namespace nginx-gateway --create-namespace
kubectl get pods,svc -n nginx-gateway     # pod CrashLoopBackOff
kubectl logs -n nginx-gateway <pod-ngf>
# no matches for kind GRPCRoute in version gateway.networking.k8s.io
kubectl api-resources | grep gateway
```

- **Nguyên nhân**: CRD bản 1.0.0 **cũ** — không có kind GRPCRoute mà controller
  yêu cầu.
- **Cách xử lý**: GitHub releases có bản **1.1.0** mới hơn → apply lại URL bản
  đó. Bài học: gặp lỗi CRD đầu tiên, **check bản release mới nhất** thay vì copy
  URL cũ.
- Sau khi apply đúng: số restart giảm dần; `kubectl delete pod -n nginx-gateway
  <pod>` đẩy nhanh → pod **Running**.
- `kubectl get gc` → GatewayClass `nginx` — controller tự tạo.
- Service của controller đang **LoadBalancer** — minikube không có load
  balancer thật → `kubectl edit svc -n nginx-gateway` đổi `type: NodePort`.

### Demo phần 2 — application và HTTPRoute

```bash
kubectl create deployment nginx-gw --image=nginx --replicas=3
kubectl expose deployment nginx-gw --port=80
kubectl apply -f httprouting.yaml       # file trong course repo
```

File `httprouting.yaml` tạo hai resource — các field đáng chú ý:

- **Gateway** `example-gateway`: `gatewayClassName: nginx`.
- **HTTPRoute**: `parentRefs` → `example-gateway`; hostname `whatever.com`;
  `backendRefs` → Service `nginx-gw`.

Test nhanh bằng port-forward — đâm thẳng vào controller, bypass Service:

```bash
sudo vim /etc/hosts                                  # 127.0.0.1 whatever.com
kubectl -n nginx-gateway port-forward svc/ngf 8080:80 8443:443
# Ctrl+Z rồi bg — port-forward chiếm console
curl whatever.com:8080                               # trang default của nginx
```

Truy cập thật qua nodePort — `/etc/hosts` trỏ `whatever.com` về minikube IP
(192.168.49.2), lấy nodePort từ service của controller:

```bash
kubectl get svc -n nginx-gateway     # nodePort demo: 31702
curl whatever.com:31702              # Welcome to nginx
```

*(Transcript ghi log lỗi dạng "kine2gpc-route-inversion-gateway-networking" —
chuẩn hóa thành GRPCRoute / gateway.networking.k8s.io; target port-forward đọc
thành "ngf-nginx-gateway" — chuẩn hóa thành svc/ngf.)*

## Phần 10 — Troubleshooting networking

Checklist đi từ pod ra phía user — kiểm từng mắt xích:

1. `kubectl get pods -o wide` — pods Running, có pod IPs.
2. `kubectl get svc` / `kubectl describe svc` — **Endpoints** có nối pod IPs.
3. Selector của Service **khớp label** pods hay không.
4. NetworkPolicy có chặn không — `kubectl get netpol -A`.
5. `kubectl describe ing` — backend có endpoints; Ingress trống endpoints →
   check Service mà nó dùng.

### Demo 1 — Service mất endpoints

**Hiện tượng:** `curl 192.168.49.2:31146` fail.

```bash
kubectl describe svc nginxsvc       # Endpoints: <none>
kubectl get pods --show-labels      # pods mang label thật
kubectl edit svc nginxsvc           # sửa selector cho khớp
curl 192.168.49.2:31146             # chạy lại được gần tức thì
```

- **Nguyên nhân**: selector của Service viết **khác hoa thường** so với label
  pods — không khớp → Service không có endpoint nào.

### Demo 2 — Ingress không chạy

**Hiện tượng:** `curl nginxsvc.info` báo fail to connect — **không phải** lỗi
name resolution (ping vẫn resolve được) → vấn đề ở phía đích.

```bash
kubectl get ing                          # Ingress vẫn tồn tại
kubectl describe ing nginxsvc           # backend IPs đầy đủ
kubectl get ns                           # không có namespace ingress-nginx
kubectl get pods -A | grep nginx         # không thấy controller nào
minikube addons list                     # ingress: disabled
```

- **Kết luận**: Ingress resource có nhưng **không có controller chạy trong
  cluster** → không có gì đứng sau lắng nghe *(nhắc lại Phần 2 — resource không
  controller là vô tác dụng)*.

> **Ghi nhớ thi CKAD**: `Endpoints: <none>` → so selector của Service với
> `--show-labels` của pods — lệch hoa thường là bẫy kinh điển.

## Phần 11 — End-of-lesson lab — Managing Incoming Traffic

### Đề bài

Tạo deployment `lab11web` chạy image nginx, 3 replicas, truy cập được qua
virtual hostname `lab11web.example.com`.

### Lời giải

**Bước 0 — bật Ingress controller** *(lab trước vừa cho thấy cluster không có
controller)*:

```bash
minikube addons enable ingress
```

**Bước 1–2 — application và Service** — Service tạo **trước** khi viết rule:

```bash
kubectl create deployment lab11web --image=nginx --replicas=3
kubectl expose deployment lab11web --port=80
```

**Bước 3 — hostname:** `/etc/hosts` thêm `<minikube-ip> lab11web.example.com`.

**Bước 4 — Ingress rule** — copy example từ help:

```bash
kubectl create ing -h | less
kubectl create ingress lab11 --rule="lab11web.example.com/=lab11web:80"
```

**Bước 5 — verify:**

```bash
kubectl describe ing lab11       # backends có pod IPs
```

**Bước 6 — pathType:** lệnh create đặt `Exact` mặc định; đổi sang `Prefix`:

```bash
kubectl explain ing.spec.rules.http.paths    # tra vị trí field
kubectl edit ing lab11                        # pathType: Exact → Prefix
```

**Bước 7 — test:**

```bash
curl lab11web.example.com        # Welcome to nginx
```

*(Transcript ghi "lab11web.com" — đề bài và /etc/hosts dùng
lab11web.example.com, chuẩn hóa theo đề.)*

## Tổng hợp lệnh Bài 11

| Lệnh | Chức năng |
|---|---|
| `minikube addons list` / `minikube addons enable ingress` | Xem / bật Ingress controller trong minikube |
| `helm upgrade --install ingress-nginx ingress-nginx --repo ... --create-namespace` | Cài ingress-nginx vào cluster bất kỳ |
| `helm install ngf oci://ghcr.io/nginx-inc/charts/nginx-gateway-fabric` | Cài Nginx Gateway Fabric controller |
| `kubectl apply -f <standard-install.yaml>` | Cài CRD Gateway API — check bản release mới nhất |
| `kubectl create ingress <tên> --rule="<host>/<path>=<svc>:<port>"` | Tạo Ingress + rules |
| `kubectl describe ing <tên>` | Xem rules + backend endpoints |
| `kubectl edit ing <tên>` | Sửa pathType / rules trực tiếp |
| `kubectl explain ing.spec.rules.http.paths` | Tra cấu trúc field khi edit |
| `kubectl get gc` | Xem GatewayClass |
| `kubectl -n <ns> port-forward svc/<svc> 8080:80` | Test cục bộ bypass Service |
| `kubectl get netpol -A` | Liệt kê NetworkPolicy khi nghi bị chặn |

> Ingress và Gateway API khép lại Module 4 — Service giờ đã đưa traffic đến pod
> bằng mọi cách: nội bộ, node, HTTP proxy. Bài 12 — **ConfigMaps và Secrets** —
> chuyển sang đưa cấu hình và dữ liệu nhạy cảm vào ứng dụng.
