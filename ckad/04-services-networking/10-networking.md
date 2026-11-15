# Bài 10 — Networking

Bài đầu của Module 4 — miền **Services and Networking** (20% đề thi). Pod nhận IP
trong pod network nhưng không truy cập được từ bên ngoài; Service là
**internal load balancer** nối hai thế giới. Bài này phủ từ mô hình mạng tổng
quan, các loại Service, DNS tích hợp đến NetworkPolicy.

## Lộ trình bài học

1. Kubernetes networking
2. Services
3. Creating Services
4. Service resources trong microservices
5. Services và DNS
6. NetworkPolicy
7. Advanced networking — Gateway API và Istio
8. End-of-lesson lab

## Phần 1 — Kubernetes networking

- Node kết nối ra **extnet** — mạng external bên ngoài cluster, nơi user request
  đi vào.
- Kubernetes không dùng mạng vật lý cho nội bộ: nó tự tạo hai mạng nội bộ bằng
  **software-defined networking**:
  - **Cluster network**: mạng nội bộ phục vụ mục đích quản trị cluster.
  - **Pod network**: mạng nơi pods chạy ứng dụng và nhận IP — pod IP **không
    truy cập được từ ngoài cluster**.
- **Service** đứng giữa hai thế giới: internal load balancer nhận traffic và
  cân bằng tải tới các pod của ứng dụng; các pod đó là **endpoints** của
  Service.
- Hai Service type chính cho CKAD:

| Type | Reachability |
|---|---|
| **ClusterIP** (mặc định) | Chỉ nội bộ cluster |
| **NodePort** | Mở thêm một **port cao** trên node (~32000) — truy cập được từ ngoài |

- Luồng request NodePort đi từ user vào pod:

```text
user ──► external DNS ──► external load balancer ──► node:NodePort ──► Service ──► pods
```

- Mô hình microservices điển hình: **frontend** dùng NodePort (đón user), **backend**
  (database...) dùng ClusterIP (chỉ nội bộ) — Phần 3.
- Cách hiện đại hơn cho HTTP: **Ingress / Gateway API** — reverse HTTP proxy tích
  hợp vào Kubernetes *(nhìn trước — chi tiết ở Bài 11)*.

*(Transcript ghi "reversed HTTP proxy" — chuẩn hóa thành reverse HTTP proxy;
"clusternet"/"podnet" — chuẩn hóa thành cluster network/pod network.)*

## Phần 2 — Services

### Service là gì

- Service là **API resource** expose một tập pods — không phải Linux service.
- Load balancing kiểu **round-robin**: mỗi request lần lượt tới một pod; đủ nhiều
  request thì tải chia đều. Không có intelligence gì thêm.
- Tập pods xác định bằng **selector khớp label** *(nhắc lại Bài 8 Phần 2 — labels
  là keo kết nối các object)*. kube-controller-manager quét liên tục: pod thêm/
  bớt thì danh sách endpoints cập nhật trong ~2–3 giây.
- Service là **decoupled resource** — sống độc lập với ứng dụng: xóa deployment
  thì Service vẫn tồn tại nhưng **mất endpoints**. Tạo/xóa Service riêng biệt,
  không ảnh hưởng pods.
- Một Service có thể chọn pods của **nhiều deployment** — nền tảng cho kỹ thuật
  canary deployments *(nhìn trước — Bài 13 Phần 3)*.

### 5 service types

| Type | Đặc điểm |
|---|---|
| **ClusterIP** | Mặc định — expose trên IP nội bộ cluster |
| **NodePort** | Port cao trên node forward vào Service — cách "primitive" để expose ra ngoài |
| **LoadBalancer** | External load balancer của public cloud — cần cài thêm |
| **ExternalName** | Redirect ở DNS level — hữu ích khi migration |
| **Headless** | Không ClusterIP, client nối thẳng pods — dùng cho StatefulSet *(nhắc lại Bài 8 Phần 5)* |

> **Ghi nhớ thi CKAD**: trong 5 type, chỉ cần thành thạo **ClusterIP** và
> **NodePort**. Ba type còn lại chỉ cần biết tên.

### Tạo Service bằng `kubectl expose`

- `kubectl expose` expose một workload đang chạy: deployment (phổ biến nhất),
  replicaset hoặc pod — cả Service cũng được. `--port` **bắt buộc**.
- `kubectl create service` là lệnh thay thế; `expose` tiện hơn vì tự copy label
  thành selector.
- Ba port phải phân biệt khi đọc manifest Service:

| Port | Thuộc về | Ý nghĩa |
|---|---|---|
| `targetPort` | container | Port ứng dụng lắng nghe trong pod |
| `port` | Service | Port Service lắng nghe trong cluster |
| `nodePort` | node | Port trên node mở ra bên ngoài (NodePort, tùy chọn) |

### Demo — expose deployment và mở NodePort

Tạo ứng dụng rồi expose bằng một lệnh:

```bash
kubectl create deployment nginxsvc --image=nginx
kubectl scale deployment nginxsvc --replicas=3
kubectl expose deployment nginxsvc --port=80
kubectl get all -l app=nginxsvc       # svc + deployment + replicaset + 3 pods
```

- `kubectl describe svc nginxsvc`: label `app=nginxsvc` = selector `app=nginxsvc`;
  **Endpoints** liệt kê IP của 3 pods; NodePort `<unset>` vì type còn ClusterIP.
- `kubectl get endpoints nginxsvc` — danh sách pod IPs Service đang trỏ tới.
- `kubectl get svc nginxsvc -o yaml` — manifest đầy đủ: `clusterIP`, `targetPort`,
  `type: ClusterIP`... (video demo bằng lệnh imperative; muốn viết YAML tay thì
  sinh từ đây).

Kiểm chứng ClusterIP là internal-only:

```bash
minikube ssh
  curl 10.111.191.21     # chạy — node thuộc cluster network
  exit
curl 10.111.191.21       # workstation: không kết nối được
```

Đổi sang NodePort để mở ra ngoài:

```bash
kubectl edit svc nginxsvc    # type: ClusterIP → NodePort
kubectl get svc nginxsvc     # PORT(S) hiện 80:31146 — nodePort cấp động
curl 192.168.49.2:31146      # chạy được từ workstation (IP minikube)
```

- Không set `nodePort` thì cluster tự cấp động trong dải 30000–32767 *(bổ sung từ
  nghiên cứu ngoài video — video chỉ nói "port cao ~32000")*.
- NodePort là lựa chọn khi expose dịch vụ **không phải HTTP** — Ingress/Gateway
  API chỉ làm việc với HTTP/HTTPS.

*(Transcript ghi "nginx svc" — chuẩn hóa thành `nginxsvc`.)*

## Phần 3 — Service resources trong microservices

- Microservices = ghép ứng dụng từ nhiều pods đảm nhiệm khác nhau; cấu trúc tối
  giản: **frontend + backend**.
- Backend (database) chỉ cần truy cập nội bộ → **ClusterIP**.
- Frontend (web server) cần truy cập từ user → **NodePort** hoặc Ingress.
- Quản lý traffic phức tạp hơn nữa → **service mesh** (Istio — Phần 7) — **Không
  nằm trong đề CKAD**; nhiều tổ chức cũng không cần.

## Phần 4 — Services và DNS

- Service tạo xong tự đăng ký vào **CoreDNS** — DNS server nội bộ của cluster.
- Chuỗi FQDN: `<service>.<namespace>.svc.<cluster-name>`, vd
  `nginxpod.elsewhere.svc.cluster.local`.
- Cùng namespace → gọi bằng short name được. **Khác namespace → phải dùng FQDN.**

### Demo — truy cập Service ở namespace khác

```bash
kubectl describe svc -n kube-system kube-dns     # CoreDNS, port 53
kubectl create ns elsewhere
kubectl run nginxpod -n elsewhere --image=nginx
kubectl expose pod nginxpod -n elsewhere --port=80    # expose được cả pod lẻ
kubectl run testpod --image=busybox -- sleep infinity
kubectl exec -it testpod -- cat /etc/resolv.conf
```

- `/etc/resolv.conf` trong pod: `nameserver 10.96.0.10` (ClusterIP của kube-dns)
  và search domain `default.svc.cluster.local` — cluster name + namespace của pod.

```bash
kubectl exec -it testpod -- wget --spider --timeout=1 nginxpod
# fail — nginxpod ở namespace khác, short name không phân giải được
kubectl exec -it testpod -- wget --spider --timeout=1 \
  nginxpod.elsewhere.svc.cluster.local
# "remote file exists"
```

> **Ghi nhớ thi CKAD**: pod không nối được Service ở namespace khác → dùng FQDN
> `<service>.<namespace>.svc.cluster.local`. Đây là dạng bài thi thường gặp.

*(Transcript ghi "1096.0.10" — chuẩn hóa thành 10.96.0.10.)*

## Phần 5 — NetworkPolicy

### Mặc định: không giới hạn gì cả

- Kubernetes mặc định **không hạn chế network traffic** — pods liên lạc tự do,
  kể cả khác namespace. Namespace isolate **resource**, không isolate **mạng**.
- Muốn hạn chế traffic phải tạo **NetworkPolicy** — và policy chỉ có hiệu lực
  nếu **CNI plugin hỗ trợ**. Video dùng **Calico**: phải start minikube với
  `--cni=calico`. Cluster đang chạy CNI khác thì `minikube delete` rồi start lại
  với option này (theo giảng viên, Weave — CNI thường thấy trong các môi trường
  demo — không hỗ trợ NetworkPolicy).

### Logic đánh giá

| Tình huống | Kết quả |
|---|---|
| Không có policy nào chọn pod | **Allow** (mặc định) |
| Policy chọn pod, rule khớp traffic | **Allow** |
| Policy chọn pod, không rule nào khớp | **Deny** |

- Rule nhận diện traffic bằng 3 identifier: **podSelector** (label của pod nguồn),
  **namespaceSelector** (cả namespace), **ipBlock** (dải IP).
- Traffic đến/đi từ **chính node** chứa pod luôn được allow.
- Policies **additive**: nhiều policy cùng chọn một pod thì hợp các rule allow —
  không có rule nào "hủy" rule khác.
- Phạm vi CKAD: netpol đơn giản theo label. Netpol áp cho cả namespace **thuộc
  phạm vi CKA**, không cần cho CKAD.

### Demo — chặn truy cập theo label

Hai pod: `nginx` (label `app=nginx`) và `busybox` (label `app=sleepy`).
NetworkPolicy chỉ cho pod có label `access=true` nối vào nginx:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: "true"
```

```bash
kubectl apply -f networkpolicy.yaml
kubectl expose pod nginx --port=80       # nếu svc nginx đã có: delete svc cũ trước
kubectl exec -it busybox -- wget --spider --timeout=1 nginx
# download timed out — busybox chưa có label access=true
kubectl describe networkpolicy access-nginx    # xem selector policy yêu cầu
kubectl label pod busybox access=true
kubectl exec -it busybox -- wget --spider --timeout=1 nginx
# "remote file exists"
```

> **Ghi nhớ thi CKAD**: đề "pod không truy cập được workload" → `kubectl describe
> networkpolicy` xem selector nó yêu cầu → gán label đó cho **pod khởi tạo
> traffic** bằng `kubectl label`. Gần như toàn bộ netpol trong đề CKAD chỉ vậy.

## Phần 6 — Advanced networking — Gateway API và Istio

Phần này để "nghe tên mà biết" — không cần học sâu để thi.

- **Gateway API**: thế hệ mới quản lý traffic vào/ra cluster bằng **custom
  resources** — ingress lẫn egress. Bổ sung những thứ Ingress thiếu: **TCP, UDP,
  gRPC**, **traffic splitting**, **mirroring**, **WebSocket**; xu hướng tích hợp
  dần với Ingress *(nhìn trước — cấu hình chi tiết ở Bài 11)*.
- **Istio**: **service mesh** — quản lý quan hệ service-to-service trong
  microservices: traffic management, security, observability. Optional, chỉ hợp
  môi trường lớn; môi trường nhỏ dùng Ingress/Gateway API là đủ — **Không nằm
  trong đề CKAD**.

## Phần 7 — End-of-lesson lab — Managing Services

### Đề bài

1. Tạo namespace `remote`.
2. Trong `remote`, chạy pod nginx tên `remote-web`.
3. Expose ứng dụng để truy cập được trên minikube host ở **port 31999**.
4. Ở namespace `default`, chạy pod `testpod` (busybox, `sleep infinity`).
5. Từ `testpod`, verify truy cập được trang default của `remote-web` bằng
   `wget --spider --timeout=1`.
6. Verify trang này truy cập được từ host ở port 31999.

### Lời giải

**Bước 1–3 — namespace, pod, Service NodePort:**

```bash
kubectl create ns remote
kubectl -n remote run remote-web --image=nginx
kubectl -n remote expose pod remote-web --port=80
kubectl -n remote edit svc remote-web
# type: ClusterIP → NodePort, thêm nodePort: 31999
kubectl -n remote get svc          # PORT(S) hiện 80:31999
```

- Quên `--port` khi expose → lỗi "could not find port" — port là bắt buộc.
- Chi tiết viết tay: giá trị type viết hoa chữ đầu (`NodePort`) nhưng key là
  `nodePort` — chữ `n` thường.

**Bước 4–5 — test từ pod khác namespace:**

```bash
kubectl run testpod --image=busybox -- sleep infinity
kubectl exec -it testpod -- wget --spider --timeout=1 \
  remote-web.remote.svc.cluster.local      # "remote file exists"
```

**Bước 6 — test từ host:**

```bash
curl 192.168.49.2:31999                   # "Welcome to nginx"
```

*(Transcript ghi "190.2168.49.2" — chuẩn hóa thành 192.168.49.2; "w get" →
`wget`.)*

## Tổng hợp lệnh Bài 10

| Lệnh | Chức năng |
|---|---|
| `kubectl expose deployment/pod <tên> --port=80 [--type=NodePort]` | Tạo Service expose pods |
| `kubectl get endpoints [<svc>]` | Xem pod IPs Service đang trỏ tới |
| `kubectl describe svc <tên>` | Xem selector, ports, endpoints |
| `kubectl get svc <tên> -o yaml` | Xem manifest Service đầy đủ |
| `kubectl edit svc <tên>` | Sửa type / nodePort trực tiếp |
| `kubectl exec -it <pod> -- cat /etc/resolv.conf` | Xem nameserver + search domains |
| `kubectl exec -it <pod> -- wget --spider --timeout=1 <FQDN>` | Test DNS + kết nối từ pod |
| `kubectl apply -f <netpol>.yaml` / `kubectl describe networkpolicy <tên>` | Tạo / xem NetworkPolicy |
| `kubectl label pod <pod> key=value` | Thêm label khớp selector |
| `minikube ssh` → `curl <ClusterIP>` | Test ClusterIP từ node |

> NodePort mở được mọi port, nhưng với HTTP có cách hiện đại hơn. Bài 11 đi vào
> **Ingress và Gateway API** — reverse HTTP proxy tích hợp sâu vào Kubernetes.
