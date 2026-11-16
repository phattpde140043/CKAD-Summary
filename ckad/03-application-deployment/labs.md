# Labs — Module 3: Application Deployment

Tổng hợp toàn bộ ví dụ demo và lab exercise từ các bài học trong module, chuyển thành checklist
thực hành kèm tiêu chí verify. Mục tiêu: làm tới khi không cần nhìn tài liệu.

## Bài 8

### Lab 8.1 — Labels, selectors và annotations (demo Phần 2)

- [ ] Tạo deployment và pod đơn lẻ, so sánh label mặc định:

```bash
kubectl create deployment myapp --image=nginx
kubectl run standalone --image=nginx
kubectl get pods --show-labels
```

**Verify:** pod của deployment mang `app=myapp`; pod standalone mang
`run=standalone`.

- [ ] Lọc theo label, thêm/xóa label trên pod:

```bash
kubectl get pods -l app=myapp
kubectl label pod standalone env=test
kubectl get pods --show-labels
kubectl label pod standalone env-
```

**Verify:** `-l` chỉ trả pod khớp; label `env` xuất hiện rồi biến mất sau
lệnh xóa (`env-`).

- [ ] Gỡ label đang khớp selector khỏi pod được quản lý:

```bash
kubectl label pod <pod-của-myapp> app-
kubectl get pods --show-labels
```

**Verify:** ReplicaSet sinh pod thay thế mới mang `app=myapp`; pod cũ vẫn
chạy nhưng thành **orphan** — không còn label `app`.

- [ ] Thao tác annotations giống label:

```bash
kubectl annotate pod standalone note=hello
kubectl describe pod standalone
```

**Verify:** dòng `note: hello` xuất hiện trong mục Annotations.

- [ ] Dọn dẹp:

```bash
kubectl delete deployment myapp
kubectl delete pod standalone
```

### Lab 8.2 — Deployment scalability (demo Phần 3)

- [ ] Tạo deployment 3 replicas rồi scale lên 6:

```bash
kubectl create deployment scaledapp --image=nginx --replicas=3
kubectl get all
kubectl scale deployment scaledapp --replicas=6
kubectl get pods
```

**Verify:** `kubectl get all` thấy deployment + replicaset + pods; sau
`scale` đủ 6 pods.

- [ ] Xóa 1 pod, quan sát pod thay thế:

```bash
kubectl delete pod <pod-bất-kỳ>
kubectl get pods
```

**Verify:** tổng vẫn 6 pods — pod bị xóa mất tên, pod thay thế mang tên mới
cùng hash ReplicaSet.

- [ ] Dọn dẹp:

```bash
kubectl delete deployment scaledapp
```

### Lab 8.3 — Rolling update và rollout history (demo Phần 4)

- [ ] Apply deployment từ course repo (strategy rollingUpdate, image nginx
  bản cũ):

```bash
kubectl apply -f rolling.yaml
kubectl set image deployment <tên> nginx=nginx:1.15
kubectl get pods
```

**Verify:** pods mới lên, pods cũ xuống dần — không lúc nào thiếu quá
`maxUnavailable`.

- [ ] Xem lịch sử rồi update tiếp bằng editor:

```bash
kubectl rollout history deployment <tên>
kubectl edit deployment <tên>        # sửa image thành nginx:1.17
kubectl rollout history deployment <tên>
```

**Verify:** mỗi lần update sinh thêm một revision.

- [ ] Rollback về revision đầu và kiểm tra ReplicaSet:

```bash
kubectl rollout undo deployment <tên> --to-revision=1
kubectl get replicasets
```

**Verify:** các ReplicaSet cũ vẫn còn với **0 replicas** — đây là "lịch sử"
cho rollback.

- [ ] Dọn dẹp:

```bash
kubectl delete deployment <tên>
```

**Ghi nhớ:** rollout chạy rất nhanh vì image nginx đã cache trên node; muốn
quan sát rõ tiến trình rolling, dùng image chưa có trên node.

### Lab 8.4 — StatefulSet và troubleshoot PVC (demo Phần 5)

- [ ] Apply StatefulSet từ course repo và gặp lỗi:

```bash
kubectl apply -f statefulset.yaml
kubectl get pods                        # web-0 treo Pending
kubectl describe pod web-0
```

**Verify:** Events hiện `pod has unbound immediate PersistentVolumeClaims`.

- [ ] Sửa file: bỏ `storageClassName` trong volumeClaimTemplates (dùng
  default StorageClass), lưu thành `statefulset-new.yaml` rồi apply:

```bash
kubectl apply -f statefulset-new.yaml
kubectl get pods                        # web-0 VẪN Pending
```

**Verify:** pod vẫn Pending — PVC cũ `www-web-0` còn nằm đó với
storageClassName sai, không tự cập nhật theo manifest mới.

- [ ] Xóa PVC cũ cho StatefulSet sinh lại:

```bash
kubectl delete pvc www-web-0
kubectl get pvc,pods
```

**Verify:** PVC mới **Bound**; pod `web-0` Running trước, `web-1` khởi động
theo sau — đúng thứ tự của StatefulSet.

- [ ] Dọn dẹp (scale về 0 trước khi delete để dọn sạch pods):

```bash
kubectl scale statefulset web --replicas=0
kubectl delete statefulset web
kubectl delete pvc www-web-1
```

### Lab 8.5 — DaemonSet (demo Phần 6)

- [ ] Sinh YAML từ deployment rồi sửa thành DaemonSet:

```bash
kubectl create deployment daemonnginx --image=nginx \
  --dry-run=client -o yaml > daemon.yaml
```

Sửa `daemon.yaml` đúng 3 chỗ: `kind: Deployment` → `kind: DaemonSet`; xóa
dòng `replicas`; xóa block `strategy`.

- [ ] Apply và kiểm tra:

```bash
kubectl apply -f daemon.yaml
kubectl get ds,pods
```

**Verify:** đúng 1 pod — minikube chỉ có 1 node (DaemonSet chạy một pod trên
mỗi node).

- [ ] Dọn dẹp:

```bash
kubectl delete daemonset daemonnginx
```

**Ghi nhớ:** không có `kubectl create daemonset` — mẹo sinh YAML bằng
`--dry-run=client -o yaml` từ deployment rồi sửa.

### Lab 8.6 — Autoscaling với HPA (demo Phần 7) — không nằm trong đề CKAD

- [ ] Bật metrics-server và dựng ứng dụng stress:

```bash
minikube addons enable metrics-server
cd autoscaling                         # thư mục trong course repo
docker build -t php-apache .
kubectl apply -f hpa.yaml               # deployment + service php-apache
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl get hpa
```

**Verify:** HPA hiện target 50% CPU, min 1 / max 10.

- [ ] Tạo tải và quan sát HPA scale lên:

```bash
kubectl run -it load-generator --rm --image=busybox --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

Terminal khác:

```bash
kubectl get hpa
kubectl get deployment php-apache
```

**Verify:** sau vài phút CPU ~250% so với ngưỡng 50%; replicas tăng (demo:
lên 5 pods).

- [ ] Dừng tải và quan sát scale xuống:

```bash
kubectl delete pod load-generator
kubectl get hpa                         # chạy lại vài lần
```

**Verify:** HPA scale **xuống chậm**, dần dần về 1 pod — không giảm một
nhát.

- [ ] Dọn dẹp:

```bash
kubectl delete hpa php-apache
kubectl delete deployment,service php-apache
minikube addons disable metrics-server
```

### Lab 8.7 — End-of-lesson lab (Phần 8)

**Đề bài:** deployment chạy `nginx:1.9`, 5 replicas, pods mang label
`type=proxy`, khi update không bao giờ thiếu quá 2 pods.

- [ ] Sinh YAML và sửa:

```bash
kubectl create deployment lab9deploy --image=nginx:1.9 --replicas=5 \
  --dry-run=client -o yaml > lab9.yaml
```

Thêm `type: proxy` vào labels của **pod template** trong `lab9.yaml`.

- [ ] Mượn block strategy từ deployment dummy:

```bash
kubectl create deployment dummy --image=nginx
kubectl get deployment dummy -o yaml    # copy block strategy về lab9.yaml
kubectl delete deployment dummy
```

Sửa trong `lab9.yaml`: `maxUnavailable: 2` ("không thiếu quá 2 pods").

- [ ] Apply và xử lý lỗi pull image:

```bash
kubectl apply -f lab9.yaml
kubectl get pods                        # ImagePullBackOff
kubectl describe pod <pod>
kubectl set image deployment lab9deploy nginx=nginx:latest
kubectl get pods --show-labels
```

**Verify:** 5 pods Running, đều mang label `type=proxy`.

- [ ] Dọn dẹp:

```bash
kubectl delete deployment lab9deploy
```

**Ghi nhớ:** kiểm tra image đã cache trên node: `minikube ssh` →
`crictl images`; image chưa cache giúp thấy rõ hành vi rolling.

## Bài 9

### Lab 9.1 — Helm binary, repository và search (demo Phần 2–3)

- [ ] Cài Helm binary (nếu chưa có — tải bản đúng platform từ
  `github.com/helm/helm` → Releases):

```bash
tar xvf helm-v*-linux-*.tar.gz
sudo mv linux-*/helm /usr/local/bin
helm version
```

**Verify:** `helm version` in phiên bản.

- [ ] Thêm repo Bitnami và tìm chart:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo list
helm search repo nginx
helm search repo nginx --versions
helm repo update
```

**Verify:** repo bitnami có trong `helm repo list`; search trả chart kèm
app version; `--versions` liệt kê cả bản cũ.

### Lab 9.2 — Cài đặt ứng dụng bằng Helm (demo Phần 4)

- [ ] Cài mysql với tên tự sinh:

```bash
helm install bitnami/mysql --generate-name
kubectl get all
helm list
helm status <tên-release>
```

**Verify:** `kubectl get all` thấy StatefulSet + pod mysql; `helm status`
hiện deployed + username + cách lấy mật khẩu từ Secret.

- [ ] Xem mô tả chart rồi gỡ release:

```bash
helm show chart bitnami/mysql
helm delete <tên-release>
helm list
```

**Verify:** `helm list` trống sau delete.

### Lab 9.3 — Helm values và upgrade (demo Phần 4)

- [ ] Xem values mặc định của chart nginx:

```bash
helm show values bitnami/nginx | grep -E 'replicaCount|commonLabels'
```

**Verify:** `replicaCount` mặc định là 1.

- [ ] Tạo file `values.yaml`:

```yaml
commonLabels:
  type: helmapp
replicaCount: 3
```

- [ ] Cài với values riêng:

```bash
helm install nginx bitnami/nginx --values values.yaml
kubectl get deploy --show-labels
```

**Verify:** deployment có 3 replicas + label `type=helmapp`.

- [ ] Upgrade KHÔNG truyền lại `--values`:

```bash
helm upgrade nginx bitnami/nginx
helm get values nginx
```

**Verify:** `helm get values` trống — values tùy chỉnh đã mất về mặc định.

- [ ] Upgrade lại có `--values`:

```bash
helm upgrade nginx bitnami/nginx --values values.yaml
helm get values nginx
helm get values nginx --all
```

**Verify:** values riêng trở lại; `--all` hiện cả defaults.

- [ ] Dọn dẹp:

```bash
helm delete nginx
```

**Ghi nhớ:** `helm upgrade` không tự nhớ `--values` — luôn truyền lại.

### Lab 9.4 — Kustomize (demo Phần 5)

- [ ] Vào thư mục demo trong course repo, đọc cấu hình:

```bash
cd kustomize
cat kustomization.yaml
```

**Verify:** thấy `resources` (deployment + service) và `namePrefix`.

- [ ] Apply và xóa bằng `-k`:

```bash
kubectl apply -k .
kubectl get all                # tên resource thêm prefix test-
kubectl delete -k .
kubectl get all
```

**Verify:** sau apply, resource mang tiền tố `test-`; sau delete không còn
gì của bộ đó.

### Lab 9.5 — End-of-lesson lab: cài Vault (Phần 6)

**Đề bài:** cài HashiCorp Vault bằng Helm theo hướng dẫn artifacthub.io.

- [ ] Lấy 2 lệnh từ artifacthub.io (tìm "Vault" → HashiCorp Vault → nút
  Install) rồi chạy:

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install my-vault hashicorp/vault
```

- [ ] Verify:

```bash
helm get manifest my-vault
kubectl get all
```

**Verify:** manifest liệt kê mọi component; `kubectl get all` thấy
StatefulSet `my-vault` + pods + services.

- [ ] Dọn dẹp:

```bash
helm delete my-vault
```

## Deep dive labs (bổ sung ngoài video)

### Lab DD.2 — So sánh chiến lược rollout

- [ ] Dựng Deployment và ép rollout "chậm rãi" để quan sát:

```bash
kubectl create deployment roll --image=nginx --replicas=4
kubectl patch deployment roll -p \
  '{"spec":{"minReadySeconds":30,"strategy":{"rollingUpdate":{"maxUnavailable":0,"maxSurge":2}}}}'
kubectl set image deployment/roll nginx=nginx:1.27-alpine
kubectl rollout status deployment/roll
```

**Verify:** trong lúc rollout, `kubectl get pods` (màn khác) không bao
giờ xuống dưới 4 pod (maxUnavailable 0), có lúc lên 6 (maxSurge 2);
mỗi pod mới phải "ngồi im" 30 giây (minReadySeconds) trước khi pod cũ
bị thay — mất vài phút là bình thường.

- [ ] Gắn change-cause cho từng revision:

```bash
kubectl annotate deployment roll \
  kubernetes.io/change-cause="v2: alpine 1.27" --overwrite
kubectl set image deployment/roll nginx=nginx:1.27
kubectl annotate deployment roll \
  kubernetes.io/change-cause="v3: quay về 1.27" --overwrite
kubectl set image deployment/roll nginx=nginx:1.27-alpine
kubectl rollout history deployment/roll
```

**Verify:** các revision mới hiện CHANGE-CAUSE rõ ràng; revision đầu
tiên `<none>` — annotation phải có **trước** lúc revision sinh ra mới
được sao chép vào ReplicaSet.

- [ ] Rollback và xem lại lịch sử:

```bash
kubectl rollout undo deployment/roll --to-revision=2
kubectl rollout history deployment/roll
```

**Verify:** undo **không** giảm số revision — sinh revision mới chứa
template của revision cũ đã chọn.

- [ ] So sánh Recreate (có downtime):

```bash
kubectl patch deployment roll -p \
  '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'
kubectl set image deployment/roll nginx=nginx:alpine
kubectl get pods -w
```

**Verify:** có khoảnh khắc đếm pod về **0** — đủ 4 pod cũ Terminating
hết rồi pod mới mới được tạo. Chú ý patch phải đặt `rollingUpdate:
null`; để lại block cũ là invalid.

- [ ] Dọn dẹp:

```bash
kubectl delete deployment roll
```

**Ghi nhớ:** maxUnavailable 0 + maxSurge > 0 = zero-downtime rollout;
Recreate chấp nhận downtime để không chạy 2 phiên bản song song.
