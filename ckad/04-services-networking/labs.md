# Labs — Module 4: Services and Networking

Tổng hợp toàn bộ ví dụ demo và lab exercise từ các bài học trong module, chuyển thành checklist
thực hành kèm tiêu chí verify. Mục tiêu: làm tới khi không cần nhìn tài liệu.

## Bài 10

### Lab 10.1 — Expose deployment và mở NodePort (demo Phần 2)

- [ ] Tạo deployment 3 replicas và expose:

```bash
kubectl create deployment nginxsvc --image=nginx
kubectl scale deployment nginxsvc --replicas=3
kubectl expose deployment nginxsvc --port=80
```

**Verify:** `kubectl get all -l app=nginxsvc` thấy Service + Deployment +
ReplicaSet + 3 pods; `kubectl get endpoints nginxsvc` liệt kê 3 pod IPs.

- [ ] Test ClusterIP nội bộ: `minikube ssh` → `curl <ClusterIP>` (chạy được);
      `exit` rồi `curl <ClusterIP>` từ workstation (không nối được).
- [ ] Mở NodePort: `kubectl edit svc nginxsvc` — `type: ClusterIP` → `NodePort`.

**Verify:** `kubectl get svc nginxsvc` hiện port động dạng `80:31xxx/TCP`;
`curl $(minikube ip):31xxx` trả về trang default của nginx.

- [ ] Dọn dẹp: `kubectl delete deployment,svc nginxsvc`

**Ghi nhớ:** không set `nodePort` thì cluster cấp động trong dải 30000–32767;
muốn port cố định, thêm `nodePort: <số>` khi edit.

### Lab 10.2 — DNS và FQDN cross-namespace (demo Phần 4)

- [ ] Tạo Service ở namespace khác:

```bash
kubectl create ns elsewhere
kubectl -n elsewhere run nginxpod --image=nginx
kubectl -n elsewhere expose pod nginxpod --port=80
```

- [ ] Tạo pod test: `kubectl run testpod --image=busybox -- sleep infinity`
- [ ] `kubectl exec -it testpod -- cat /etc/resolv.conf`

**Verify:** nameserver là ClusterIP của kube-dns (vd 10.96.0.10) + search
domain `default.svc.cluster.local`.

- [ ] Short name: `kubectl exec -it testpod -- wget --spider --timeout=1 nginxpod`
      → fail (Service ở namespace khác).
- [ ] FQDN: `kubectl exec -it testpod -- wget --spider --timeout=1
      nginxpod.elsewhere.svc.cluster.local`

**Verify:** output "remote file exists".

- [ ] Dọn dẹp: `kubectl delete ns elsewhere; kubectl delete pod testpod`

**Ghi nhớ:** khác namespace phải dùng FQDN `<service>.<namespace>.svc.cluster.local`.

### Lab 10.3 — NetworkPolicy chặn theo label (demo Phần 5)

Prereq: cluster chạy CNI hỗ trợ NetworkPolicy — `minikube start --cni=calico`;
check `kubectl get pods -n kube-system` thấy calico pods.

- [ ] Tạo pods: `kubectl run nginx --image=nginx` +
      `kubectl label pod nginx app=nginx --overwrite` +
      `kubectl run busybox --image=busybox -- sleep infinity`
- [ ] Tạo NetworkPolicy `access-nginx`: podSelector `app=nginx`, ingress từ
      podSelector `access: "true"` (YAML trong Phần 5) → `kubectl apply -f`
- [ ] `kubectl expose pod nginx --port=80` (nếu svc `nginx` cũ còn: xóa trước)
- [ ] Test bị chặn: `kubectl exec -it busybox -- wget --spider --timeout=1 nginx`

**Verify:** "download timed out" — busybox chưa có label `access=true`.

- [ ] `kubectl describe networkpolicy access-nginx` — thấy selector yêu cầu.
- [ ] Mở: `kubectl label pod busybox access=true` rồi test lại bước trước.

**Verify:** "remote file exists".

- [ ] Dọn dẹp: `kubectl delete netpol access-nginx; kubectl delete pod nginx
      busybox; kubectl delete svc nginx`

**Ghi nhớ:** netpol xét label của **pod khởi tạo traffic** — describe policy để
biết cần gán label gì.

### Lab 10.4 — Managing Services (end-of-lesson lab)

- [ ] `kubectl create ns remote`
- [ ] `kubectl -n remote run remote-web --image=nginx`
- [ ] `kubectl -n remote expose pod remote-web --port=80`
- [ ] `kubectl -n remote edit svc remote-web` — `type: NodePort` +
      `nodePort: 31999`

**Verify:** `kubectl -n remote get svc` hiện `80:31999/TCP`.

- [ ] `kubectl run testpod --image=busybox -- sleep infinity`
- [ ] `kubectl exec -it testpod -- wget --spider --timeout=1
      remote-web.remote.svc.cluster.local`
- [ ] `curl $(minikube ip):31999`

**Verify:** bước test trong pod ra "remote file exists"; curl từ host trả về
"Welcome to nginx".

- [ ] Dọn dẹp: `kubectl delete ns remote; kubectl delete pod testpod`

**Ghi nhớ:** giá trị type là `NodePort` (chữ N hoa) nhưng key `nodePort` viết
thường — dễ gõ nhầm lúc edit.

## Bài 11

### Lab 11.1 — Cài ingress-nginx bằng Helm (demo Phần 3)

- [ ] Cài controller vào namespace riêng:

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

- [ ] Verify: `kubectl get pods -n ingress-nginx`

**Verify:** pod ingress-nginx-controller Running (0/1 Ready lúc đầu là bình
thường).

### Lab 11.2 — Bật minikube ingress addon và xử lý pod Pending (demo Phần 4)

- [ ] `minikube addons list` → ingress disabled
- [ ] `minikube addons enable ingress` (mất ~1 phút)
- [ ] Nếu treo "verifying ingress addon":

```bash
kubectl get ns
kubectl get all -n ingress-nginx
kubectl -n ingress-nginx describe pod ingress-nginx-controller-<suffix>
```

**Verify:** Events hiện `Insufficient cpu ... no preemption candidates` nếu
node thiếu CPU; tăng CPU node (demo 2 → 8 cores) → pod Pending →
ContainerCreating → Running.

**Ghi nhớ:** Pending do requests CPU vượt node — hai lối: sửa requests hoặc
thêm CPU.

### Lab 11.3 — Ingress đầu tiên (demo Phần 5)

Prereq: deployment `nginxsvc` + Service từ Bài 10 (`kubectl get all -l
app=nginxsvc`).

- [ ] `kubectl create ing -h | less` — xem syntax + example
- [ ] `kubectl create ingress nginxsvc --rule="/=nginxsvc:80"`
- [ ] `sudo vim /etc/hosts` — thêm `<minikube-ip> nginxsvc.info`
- [ ] `kubectl describe ing nginxsvc`

**Verify:** Hosts `*`, backend nginxsvc:80 kèm pod IPs; `curl nginxsvc.info`
trả về trang default của nginx.

### Lab 11.4 — Multi-host ingress và pathType (demo Phần 6)

- [ ] Tạo 2 app + expose:

```bash
kubectl create deployment mars --image=nginx
kubectl create deployment saturn --image=nginx
kubectl expose deployment mars --port=80
kubectl expose deployment saturn --port=80
```

- [ ] `/etc/hosts`: 2 dòng mars.example.com + saturn.example.com → minikube
      IP; `ping` test
- [ ] `kubectl create ingress multihost \
      --rule="mars.example.com/=mars:80" \
      --rule="saturn.example.com/=saturn:80"`
- [ ] `kubectl edit ing multihost` — pathType Exact → Prefix
- [ ] Test: curl cả 2 host OK; `curl bogus.example.com` lỗi name resolution

**Verify:** mars và saturn đều trả trang default của nginx; host không khai
báo fail ngay ở resolution.

**Ghi nhớ:** path bắt buộc, host optional trong `--rule`; `create ingress`
sinh pathType Exact mặc định.

### Lab 11.5 — Gateway API end-to-end (demo Phần 9)

Prereq: cảnh báo Phần 1 — không chạy Ingress và Gateway API trên cùng máy.

- [ ] Cài CRD (check bản release mới nhất trước):

```bash
kubectl apply -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

- [ ] Cài controller: `helm install ngf \
      oci://ghcr.io/nginx-inc/charts/nginx-gateway-fabric \
      --namespace nginx-gateway --create-namespace`
- [ ] Nếu pod CrashLoopBackOff: `kubectl logs -n nginx-gateway <pod>` — thấy
      lỗi thiếu kind GRPCRoute → apply CRD bản mới hơn (1.1.0), rồi
      `kubectl delete pod` để restart nhanh
- [ ] Verify: `kubectl get gc` thấy GatewayClass `nginx`; pod Running
- [ ] Service controller đang LoadBalancer → `kubectl edit svc -n
      nginx-gateway` đổi `type: NodePort`
- [ ] Tạo app: `kubectl create deployment nginx-gw --image=nginx
      --replicas=3` + `kubectl expose deployment nginx-gw --port=80`
- [ ] Apply `httprouting.yaml` từ course repo — check `gatewayClassName:
      nginx`, `parentRefs`, hostname `whatever.com`, `backendRefs` → nginx-gw
- [ ] Test port-forward: `/etc/hosts` thêm `127.0.0.1 whatever.com` →
      `kubectl -n nginx-gateway port-forward svc/ngf 8080:80 8443:443`
      (Ctrl+Z + `bg`) → `curl whatever.com:8080`
- [ ] Test thật: `/etc/hosts` trỏ `whatever.com` về minikube IP → lấy
      nodePort (`kubectl get svc -n nginx-gateway`) → `curl
      whatever.com:<nodePort>`

**Verify:** curl qua port-forward và qua nodePort đều trả về trang default
của nginx.

### Lab 11.6 — Troubleshooting networking (demo Phần 10)

- [ ] Kịch bản 1 — Service mất endpoints: `curl <minikube-ip>:31146` fail →
      `kubectl describe svc nginxsvc` thấy `Endpoints: <none>` → `kubectl get
      pods --show-labels` so label → `kubectl edit svc nginxsvc` sửa selector
      → curl lại OK
- [ ] Kịch bản 2 — thiếu controller: `curl nginxsvc.info` fail (ping vẫn
      resolve) → `kubectl describe ing` backend đầy đủ → `kubectl get pods -A
      | grep nginx` không thấy controller → `minikube addons list` → ingress
      disabled → enable

**Verify:** sau sửa selector curl chạy lại gần tức thì; sau khi enable
ingress, curl nginxsvc.info trả trang nginx.

**Ghi nhớ:** mô tả lỗi phân biệt nguyên nhân — lỗi name resolution ≠ lỗi
"fail to connect".

### Lab 11.7 — Managing Incoming Traffic (end-of-lesson lab)

- [ ] `minikube addons enable ingress`
- [ ] `kubectl create deployment lab11web --image=nginx --replicas=3`
- [ ] `kubectl expose deployment lab11web --port=80`
- [ ] `/etc/hosts`: `<minikube-ip> lab11web.example.com`
- [ ] `kubectl create ingress lab11 --rule="lab11web.example.com/=lab11web:80"`
- [ ] `kubectl describe ing lab11` — backends có pod IPs
- [ ] `kubectl explain ing.spec.rules.http.paths` → `kubectl edit ing lab11`
      đổi pathType Exact → Prefix

**Verify:** `curl lab11web.example.com` trả về "Welcome to nginx".

**Ghi nhớ:** Service tạo trước khi viết rule; pathType mặc định của `create
ingress` là Exact.

## Deep dive labs (bổ sung ngoài video)

### Lab DD.8 — Mô phỏng sự cố Ingress TLS (production incident)

Prereq: minikube ingress addon đang chạy *(như Lab 11.2)*; thêm vào
`/etc/hosts` một dòng: `<minikube-ip> shop.local bogus.local`.

- [ ] Dựng ứng dụng + cert + Ingress TLS:

```bash
kubectl create deployment ddweb --image=nginx --replicas=2
kubectl expose deployment ddweb --port=80
openssl req -x509 -newkey rsa:2048 -nodes -days 1 \
  -keyout tls.key -out tls.crt -subj "/CN=shop.local"
kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key
kubectl create ingress ddshop --class=nginx \
  --rule="shop.local/=ddweb:80,tls=shop-tls"
kubectl get ing ddshop
```

**Verify:** PORTS hiện **80, 443**; `curl -k https://shop.local` trả
trang chào của nginx — TLS đã terminate tại controller.

- [ ] Đọc hạn cert như on-call (không cần vào controller):

```bash
kubectl get secret shop-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -dates
```

**Verify:** notAfter = ngày mai (cert chỉ sống 1 ngày theo `-days 1`).

- [ ] Sự cố "cert hết hạn" → rotate bằng Secret cùng tên:

```bash
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout tls.key -out tls.crt -subj "/CN=shop.local"
kubectl delete secret shop-tls
kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key
```

- [ ] Xác nhận controller đã nhận cert mới:

```bash
openssl s_client -connect shop.local:443 -servername shop.local \
  </dev/null 2>/dev/null | openssl x509 -noout -dates
```

**Verify:** notAfter nhảy sang **năm sau** — ingress-nginx watch Secret
và reload, không cần đụng vào Ingress.

- [ ] Bẫy defaultBackend:

```bash
curl -k https://bogus.local
```

**Verify:** request vẫn tới controller (host có trong /etc/hosts)
nhưng không khớp rule nào → controller trả trang lỗi mặc định của nó,
không phải từ ddweb.

- [ ] Dọn dẹp:

```bash
kubectl delete ing ddshop
kubectl delete secret shop-tls
kubectl delete deploy,svc ddweb
rm tls.crt tls.key
```

Xóa dòng `shop.local bogus.local` khỏi `/etc/hosts`.

**Ghi nhớ:** nghi vấn cert → decode Secret + `openssl x509 -noout
-dates`; rotate = thay Secret cùng tên, Ingress giữ nguyên.

