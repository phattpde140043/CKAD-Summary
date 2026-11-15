# Labs — Module 7: Sample Exam

Tổng hợp toàn bộ ví dụ demo và lab exercise từ các bài học trong module, chuyển thành checklist
thực hành kèm tiêu chí verify. Mục tiêu: làm tới khi không cần nhìn tài liệu.

## Bài 18

Prereq: reset Minikube trước khi bắt đầu — `minikube stop` →
`minikube delete` → `minikube start --cni=calico`; verify `kubectl get
all` sạch. Đặt timer 2 giờ và làm cả 15 task liên tục như thi thật.

### Lab 18.1 — Namespace, pod và Secret (Task 1)

- [ ] Namespace `indiana`; secret `insecret` lưu biến `color=blue`; pod
      `inpod` chạy `nginx:latest` trong namespace này:

```bash
kubectl create ns indiana
kubectl create secret generic insecret --from-literal=color=blue -n indiana
```

- [ ] `kubectl set env` không chạy được trên pod → mượn deployment:
      tạo deploy tạm, set env từ secret, copy env block vào `task1.yaml`
      của pod `inpod` rồi apply:

```bash
kubectl create deploy task1 --image=nginx -n indiana
kubectl -n indiana set env --from=secret/insecret deploy/task1
kubectl edit deploy task1 -n indiana    # copy env block
kubectl apply -f task1.yaml
```

**Verify:** pod inpod Running; biến môi trường của pod nhận giá trị
blue từ secret.

**Ghi nhớ:** `set env` chỉ work với deployment — pod thì phải đi qua
YAML.

### Lab 18.2 — Tìm pods theo label (Task 2)

- [ ] Ghi danh sách **chỉ gồm tên** các pods có label
      `tier=control-plane` vào file `/tmp/task2pods`:

```bash
kubectl get pods -A --selector=tier=control-plane | awk '{print $2}' \
  > /tmp/task2pods
```

- [ ] Xóa dòng header (NAMESPACE) — file chỉ được chứa tên pod.

**Verify:** file chứa đúng tên các pod control-plane và không có gì
khác.

### Lab 18.3 — ConfigMap mount vào pod (Task 3)

- [ ] File `index.html` với nội dung `Welcome to the task3 web server`;
      ConfigMap `task3cm` chứa nội dung file này:

```bash
echo Welcome to the task3 web server > index.html
kubectl create cm task3cm --from-file=index.html
```

- [ ] Pod `oregonpod` chạy `nginx:latest`, mount configMap vào
      `/usr/share/nginx/html` — lấy YAML mẫu từ docs "Configure a Pod
      to use a ConfigMap" → "Add ConfigMap data to a volume"; **xóa
      dòng `restartPolicy: Never`** khi paste vào task3.yaml.

**Verify:** pod oregonpod 1/1 Running; `kubectl exec -it oregonpod --
cat /usr/share/nginx/html/index.html` trả đúng nội dung.

### Lab 18.4 — Sidecar (Task 4)

- [ ] Pod `sidepod`: container chính nginx:latest + sidecar container
      (alpine) chạy command `sleep 15` — theo docs "Sidecar
      containers": sidecar là init container với
      `restartPolicy: always`.

**Verify:** pod sidepod 2/2 Running; sidecar restart mỗi 15 giây.

**Ghi nhớ:** pod thường có apiVersion `v1` — nếu paste từ mẫu
`apps/v1` thì apply sẽ lỗi.

### Lab 18.5 — Liveness probe (Task 5)

- [ ] Namespace `probes`; pod `probepod` chạy busybox với command
      `sleep infinity`
- [ ] Liveness probe check healthz endpoint trên kubernetes service

**Verify:** pod giữ trạng thái Running — probe pass liên tục.

### Lab 18.6 — Deployment và rolling update (Task 6)

- [ ] Deployment `updates`: image `nginx:1.17`, 3 replicas, label
      `type=prod`, strategy RollingUpdate maxSurge 50% / maxUnavailable
      50% (phần `rollingUpdate` cùng cấp indent với `type`):

```bash
kubectl create deploy updates --image=nginx:1.17 --replicas=3 \
  --dry-run=client -o yaml > task6.yaml
```

- [ ] Update image lên `nginx:latest`, sau đó roll back:

```bash
kubectl apply -f task6.yaml
kubectl set image deploy/updates nginx=nginx:latest
kubectl rollout status deploy/updates
kubectl rollout undo deploy/updates
```

**Verify:** sau rollback deployment chạy lại `nginx:1.17`; strategy
vẫn 50%/50%.

### Lab 18.7 — Service và Ingress (Task 7)

- [ ] `minikube ip` lấy IP; thêm dòng `<ip> myapp.info` vào
      `/etc/hosts` (mẫu: `192.168.49.2 myapp.info`)
- [ ] Service `task7svc` cho ứng dụng updates; Ingress host
      `myapp.info`:

```bash
kubectl expose deploy updates --name=task7svc --port=80
minikube addons enable ingress
kubectl create ing myapp --rule=myapp.info/=task7svc:80
```

- [ ] Test:

```bash
curl myapp.info
```

**Verify:** curl myapp.info trả trang nginx của ứng dụng updates.

### Lab 18.8 — NetworkPolicy (Task 8)

- [ ] Pod `nevaginx` chạy `nginx:latest`, label `type=webapp`; expose
      pod (wget theo tên cần service):

```bash
kubectl run nevaginx --image=nginx:latest
kubectl label pod nevaginx type=webapp
kubectl expose pod nevaginx --port=80
```

- [ ] Pod `nevatest` chạy `busybox:latest`, label `type=tester`;
      NetworkPolicy (task8.yaml) chỉ cho pod label `type=tester` access
      tới nevaginx; test:

```bash
kubectl run nevatest --image=busybox:latest -- sleep infinity
kubectl label pod nevatest type=tester
kubectl apply -f task8.yaml
kubectl exec -it nevatest -- wget --spider --timeout=1 nevaginx
```

**Verify:** wget thành công (remote file exists).

**Ghi nhớ:** cần CNI hỗ trợ NetworkPolicy — minikube phải start với
`--cni=calico`.

### Lab 18.9 — hostPath volume (Task 9)

- [ ] Trên minikube host:

```bash
minikube ssh
sudo mkdir /webapp
sudo sh -c 'echo Welcome to the store > /webapp/index.html'
exit
```

- [ ] Pod `storepod` chạy `nginx:latest`, hostPath `/webapp` mount vào
      `/usr/share/nginx/html`; label `type=storage` rồi expose NodePort
      và sửa nodePort 32032:

```bash
kubectl label pod storepod type=storage
kubectl expose pod storepod --type=NodePort --port=80
kubectl edit svc storepod    # thêm nodePort: 32032
curl $(minikube ip):32032
```

**Verify:** curl trả về text `Welcome to the store`.

**Ghi nhớ:** field là `nodePort` — n thường, P hoa; nằm trong port
entry với đúng indent.

### Lab 18.10 — Helm install MySQL (Task 10)

- [ ] Helm install chart mysql của Bitnami:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install bitnami/mysql --generate-name
```

**Verify:** release mysql deployed (triển khai StatefulSet).

### Lab 18.11 — Memory requests và limits (Task 11)

- [ ] Namespace `nebraska`; deployment `snowdeploy` chạy
      `nginx:latest`; set memory request 64Mi, limit 128Mi:

```bash
kubectl create ns nebraska
kubectl create deploy snowdeploy --image=nginx:latest -n nebraska
kubectl set resources -n nebraska deploy/snowdeploy \
  --requests=memory=64Mi --limits=memory=128Mi
```

**Verify:** describe deployment thấy request 64Mi / limit 128Mi.

### Lab 18.12 — Canary deployment (Task 12)

- [ ] Namespace `birds`; deployment `oldbird` (`nginx:1.17`) +
      deployment `newbird` (`nginx:latest`), label `type=oldbirds` ở
      cả deployment lẫn pod template; Service `oldbirds` selector
      `type=oldbirds`, NodePort 32323:

```bash
kubectl create ns birds
kubectl -n birds expose deploy oldbird --name=oldbirds \
  --selector=type=oldbirds --type=NodePort --port=80
kubectl -n birds edit svc oldbirds    # nodePort: 32323
```

- [ ] Load balance 20% requests tới newbird, 80% tới oldbird — dùng
      số replicas:

```bash
kubectl -n birds scale deploy/oldbird --replicas=4
```

**Verify:** truy cập được service qua port 32323; 4 pods oldbird + 1
pod newbird = ~80% requests về oldbird.

### Lab 18.13 — Security context (Task 13)

- [ ] Pod `securepod` chạy `nginx:latest`: containers không được
      escalate privileges; start với supplemental group GID 2000

**Verify:** pod Running; securityContext đáp ứng cả 2 yêu cầu.

### Lab 18.14 — Build và export image (Task 14)

- [ ] Build image `myapp:1.0` từ Dockerfile trong course Git repo
      (dùng Dockerfile nào cũng được — task là về create image); export
      ra `/tmp/myapp.tar`:

```bash
find . -name dockerfile
docker build -t myapp:1.0 .
docker save myapp:1.0 > /tmp/myapp.tar
file /tmp/myapp.tar
```

**Verify:** file `/tmp/myapp.tar` tồn tại (đúng là tar ball).

**Ghi nhớ:** docker image mặc định đã OCI compliant — yêu cầu "OCI
compliant" được thỏa miễn phí; `docker save -o file image` hoặc shell
redirection `>` đều được.

### Lab 18.15 — ServiceAccount (Task 15)

- [ ] Namespace `oklahoma`; service account `secure`; pod `securepod`
      chạy `nginx:latest` dùng serviceAccount `secure`:

```bash
kubectl create ns oklahoma
kubectl run securepod -n oklahoma --image=nginx \
  --dry-run=client -o yaml > task15.yaml
```

- [ ] Không có command-line option để set serviceAccount → thêm
      `serviceAccount: secure` vào pod spec trong `task15.yaml`; tạo
      service account trước rồi apply:

```bash
kubectl create -n oklahoma sa secure
kubectl apply -f task15.yaml
kubectl get pods -n oklahoma -o yaml | less    # serviceAccount: secure
```

**Verify:** pod securepod -n oklahoma Running với serviceAccount
secure.

**Ghi nhớ:** field là `serviceAccount` — gõ `secureaccount` thì apply
fail với lỗi "unknown field".

**Sau khi làm hết 15 task:** chạy `./examgrade.sh` trong course Git
repo (github.com/sandervanvugt/ckad) để chấm điểm — tổng **320**
điểm, instructor tự làm đạt **280** vì vài lỗi nhỏ; sau đó đối chiếu
solutions (Phần 5 bài 18).
