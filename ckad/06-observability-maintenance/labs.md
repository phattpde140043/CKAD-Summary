# Labs — Module 6: Observability & Maintenance

Tổng hợp toàn bộ ví dụ demo và lab exercise từ các bài học trong module, chuyển thành checklist
thực hành kèm tiêu chí verify. Mục tiêu: làm tới khi không cần nhìn tài liệu.

## Bài 16

### Lab 16.1 — CrashLoopBackOff nhưng exit code 0 (demo Phần 2)

- [ ] Tạo deployment và quan sát:

```bash
kubectl create deploy failureone --image=busybox
kubectl get pods
kubectl describe pod failureone-<suffix>
```

**Verify:** CrashLoopBackOff; state terminated, reason completed — bash
shell không nối standard output nên thoát ngay với exit code 0.

- [ ] Hiểu hướng khắc phục: start kèm command hoặc đặt
      `restartPolicy: Never`
- [ ] Dọn dẹp: `kubectl delete deploy failureone`

**Ghi nhớ:** exit code 0 + CrashLoopBackOff = shell thoát ngay, không
phải bug ứng dụng.

### Lab 16.2 — Exit code 1, đọc logs ứng dụng (demo Phần 2)

- [ ] Tạo deployment và quan sát:

```bash
kubectl create deploy failuretwo --image=mariadb
kubectl get pods
kubectl describe pod failuretwo-<suffix>
kubectl logs failuretwo-<suffix>
```

**Verify:** ContainerCreating chuyển CrashLoopBackOff; describe cho thấy
reason error, **exit code 1**; logs hiện lỗi ứng dụng (mariadb đòi
`MARIADB_ROOT_PASSWORD`).

**Ghi nhớ:** exit code ≠ 0 → `kubectl logs`; troubleshooting bắt đầu từ
pod, không phải deployment.

### Lab 16.3 — Sự cố chuỗi DNS → ingress → service (demo Phần 3)

Prereq: service (nginx, ClusterIP) và ingress host `myapp.local` đã chạy
trong cluster Minikube.

- [ ] Kiểm tra từng mắt xích:

```bash
curl myapp.local
ping myapp.local
kubectl get ing
kubectl describe ing myingress
kubectl get svc
```

- [ ] Curl thẳng ClusterIP của service từ node trong cluster — thành công
- [ ] Tìm ingress controller: `kubectl get ns` và
      `kubectl get all -n kube-system` — không có
- [ ] `minikube addons list` — ingress disabled
- [ ] `minikube addons enable ingress`; chờ khởi động; `curl myapp.local`

**Verify:** curl myapp.local trả response sau khi enable ingress add-on.

**Ghi nhớ:** DNS ổn + service reachable mà vẫn không vào được → nghi
ingress controller.

### Lab 16.4 — NetworkPolicy chặn traffic (demo Phần 3)

Prereq: pod nginx (`app=nginx`), pod busybox (`app=sleepy`) và
NetworkPolicy `access-nginx` chỉ cho ingress từ label `access=true`.

- [ ] `kubectl exec -it busybox -- wget --spider --timeout=1 nginx` —
      connecting + download timed out
- [ ] `kubectl get networkpolicy`; `kubectl describe networkpolicy
      access-nginx`
- [ ] `kubectl get pods --show-labels` — busybox thiếu `access=true`
- [ ] `kubectl label pod busybox access=true` — thử lại wget

**Verify:** wget thành công ("remote file exists") sau khi gán label.

**Ghi nhớ:** name resolution OK + timed out = dấu hiệu NetworkPolicy;
NetworkPolicy sống bằng labels.

### Lab 16.5 — Kiểm tra network add-on (demo Phần 3)

- [ ] `kubectl get all -n kube-system` — tìm calico-node (DaemonSet) và
      calico-kube-controllers (Deployment)
- [ ] Nếu thiếu calico: `minikube delete` rồi `minikube start
      --cni=calico`

**Verify:** calico-node + calico-kube-controllers chạy trong
kube-system.

**Ghi nhớ:** flannel không hỗ trợ NetworkPolicy — calico là lựa chọn an
toàn.

### Lab 16.6 — Cluster event logs (demo Phần 4)

- [ ] `kubectl get events`
- [ ] `kubectl get events -o wide`
- [ ] Đếm số lượng: `kubectl get events | wc -l`

**Verify:** thấy danh sách event gần đây của toàn cluster.

**Ghi nhớ:** không biết sự cố ở resource nào → `get events`; biết rồi →
`describe`.

### Lab 16.7 — Debug trong pod qua /proc (demo Phần 6)

Prereq: một pod nginx đang Running (exec chỉ có nghĩa với pod đang chạy).

- [ ] Mở shell: `kubectl exec -it nginx -- sh`
- [ ] Thử `ps` — not found; `ls` — có
- [ ] `ls /proc` — các thư mục số là PID; số cao nhất = command mới nhất
- [ ] `cat /proc/1/cmdline` — thấy process chính của container

**Verify:** cmdline của PID 1 cho thấy nginx master process.

**Ghi nhớ:** container tối giản, thiếu utilities — `/proc` là kế hoạch B
bằng Linux skills.

### Lab 16.8 — End-of-lesson lab: lab16break.sh

- [ ] Chạy script phá môi trường: `./lab16break.sh` (có trong Git repo
      của course)
- [ ] `kubectl get all` — lỗi
      `couldn't get current server API group list ... localhost:8080`
- [ ] Mở `~/.kube/config` — dòng `current-context` đang bị comment
- [ ] Bỏ dấu `#`, lưu lại; `kubectl get all` hoạt động trở lại

**Verify:** `kubectl get all` trả về resources bình thường.

**Ghi nhớ:** lỗi nhắc localhost:8080 = client mất current context —
authentication issue nằm ở `.kube/config`.

## Bài 17

### Lab 17.1 — Test API health endpoints (demo Phần 2)

- [ ] Test endpoint health của kube-apiserver (biến `$minikubeip` thay
      bằng IP thật của Minikube):

```bash
curl -k https://$minikubeip:8443/healthz
```

- [ ] So sánh với path không tồn tại:

```bash
curl -k https://$minikubeip:8443/bogus
```

**Verify:** `/healthz` trả `OK`; path `bogus` trả forbidden.

**Ghi nhớ:** kube-apiserver có 3 endpoints — `/healthz` (healthy),
`/livez` (alive), `/readyz` (ready phục vụ requests).

### Lab 17.2 — Readiness probe với file không tồn tại (demo Phần 3)

- [ ] Apply file demo (course git repo) và quan sát:

```bash
kubectl apply -f busybox-ready.yaml
kubectl get pods                          # Running nhưng 0/1
kubectl describe pod busybox-ready        # Readiness probe failed
```

- [ ] Tạo file để probe pass:

```bash
kubectl exec -it busybox-ready -- touch /tmp/nothing
kubectl get pods                          # 1/1
```

**Verify:** cột READY chuyển 0/1 → 1/1 sau khi touch file.

**Ghi nhớ:** Running ≠ Ready — luôn nhìn cột READY; readiness fail chỉ
rút pod khỏi endpoints, không restart.

### Lab 17.3 — End-of-lesson lab: probes cho pod probed

- [ ] Viết YAML pod `probed`, image `nginx:latest`, probes theo 4 điều
      kiện: 30 giây để start (startupProbe); sau khi start mỗi 10 giây
      check healthy và restart nếu fail (livenessProbe,
      periodSeconds: 10); probe test access tới Kubernetes healthz
      endpoint
- [ ] Tra documentation trang **Configure Liveness, Readiness and
      Startup Probes** — lấy example làm điểm xuất phát
- [ ] `kubectl apply -f <file.yaml>` rồi `kubectl get pods` — chờ qua
      các timeout ban đầu

**Verify:** pod chuyển 1/1 Running và giữ nguyên qua các chu kỳ probe.

**Ghi nhớ:** `exec` + `wget --no-check-certificate --spider` là cách
viết probe đơn giản nhất — chạy được Linux command là viết được probe.

## Deep dive labs (bổ sung ngoài video)

### Lab DD.3 — Metrics & events troubleshooting

Prereq: metrics-server: `minikube addons enable metrics-server`.

- [ ] Xem metrics hiện tại:

```bash
kubectl top pods
kubectl top pods --sort-by=memory
kubectl top nodes
```

**Verify:** bảng CPU(Mi)/MEMORY của pods và nodes; pod ăn RAM nhất đứng
đầu danh sách sau sort.

- [ ] Xem events theo thứ tự thời gian:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

**Verify:** events xếp cũ → mới.

- [ ] Gây lỗi rồi nhận diện:

```bash
kubectl delete deploy -n kube-system metrics-server
kubectl top pods
```

**Verify:** `error: Metrics API not available` — gặp lỗi này là tra ngay
`kubectl get apiservices | grep metrics`.

- [ ] Khôi phục: `minikube addons enable metrics-server` lần nữa; chờ
      1–2 phút; `kubectl top pods` chạy lại

**Verify:** metrics trả về bình thường.

- [ ] Dọn dẹp: giữ nguyên addon cho lab sau

**Ghi nhớ:** `kubectl top` sống chết cùng metrics-server — không phải
tính năng lõi của API server.

### Lab DD.4 — kubectl debug với pod distroless

- [ ] Tạo pod không có shell:

```bash
kubectl run pausepod --image=registry.k8s.io/pause
kubectl exec -it pausepod -- sh
```

**Verify:** exec fail — `executable file not found` (image pause không
có shell).

- [ ] Mang toolbox đến pod:

```bash
kubectl debug -it pausepod --image=busybox:1.28 --target=pausepod
```

**Verify:** shell busybox mở; `ps` thấy process `/pause` của container
gốc (process namespace sharing).

- [ ] Trong shell debug: `ls /proc/<PID-của-pause>/root` — filesystem của
      container gốc hiện ra qua `/proc`
- [ ] Kiểm tra dấu vết:

```bash
kubectl get pod pausepod \
  -o jsonpath='{.spec.ephemeralContainers[*].name}'
```

- [ ] Debug node: `kubectl get nodes` lấy tên; `kubectl debug node/<tên>
      -it --image=busybox:1.28`; `chroot /host` rồi `ls`

**Verify:** thấy filesystem thật của node.

- [ ] Dọn dẹp: `kubectl delete pod pausepod` — cách duy nhất gỡ
      ephemeral container

**Ghi nhớ:** exec cần shell trong image; `kubectl debug` mang công cụ đến
pod — distroless không còn là ngõ cụt.
