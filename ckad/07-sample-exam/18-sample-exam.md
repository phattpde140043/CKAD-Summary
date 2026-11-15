# Bài 18 — Sample Exam

Bài cuối của course — **sample exam**: exam tips cho kỳ thi thật, dựng lại
môi trường Minikube sạch, rồi **15 task** mô phỏng đề thi thực hiện trong
**2 giờ** — sau đó chấm bằng script **examgrade.sh** và đối chiếu lời giải.

## Lộ trình bài học

1. Generic exam tips
2. Preparing the sample exam environment
3. Working through the 15 sample exam tasks
4. Grading with examgrade.sh
5. Solutions

## Phần 1 — Generic exam tips

- Thi trên **máy tính của chính mình** qua **PSI Secure Browser** — chạy
  được trên mọi OS nhưng nên dùng **Windows hoặc macOS**; đừng thử trên
  Linux (support bị bỏ lại phía sau vì tổ chức phía sau không tập trung
  vào Linux). Khi chạy browser này phải **đóng toàn bộ applications** →
  không truy cập được bất cứ thứ gì trên máy mình.
- Đề thi chạy trên **live environment Ubuntu Linux** — cũng là lý do
  course khuyên dùng Minikube trên Ubuntu ngay từ đầu: quen Linux command
  line là lợi thế thật, giúp dùng Linux tools để **verify** mọi thứ.
- Mọi tools cần thiết đều **pre-installed**: kubectl, helm, Kubernetes
  cluster environment — hướng dẫn nằm trong chính đề thi. Không cần cài
  đặt gì trên Linux: người ra đề muốn exam chỉ về Kubernetes, không
  phụ thuộc Linux knowledge.
- **Làm câu theo thứ tự bất kỳ** — các câu không phụ thuộc nhau. Gặp câu
  bí ("oh my goodness, what am I supposed to do here") → **bỏ qua,
  chuyển sang câu kế**; làm trước những câu biết làm, để câu khó làm
  cuối cùng vì có thể mất rất nhiều thời gian ở đó.
- Tổng thời gian **2 giờ** — không nhiều nhưng khả thi; càng tự tin với
  các topics trong đề càng dễ hoàn thành trong khung 2 giờ.
- Đề rất cụ thể về **nơi** làm task — **đọc kỹ**. Ở đầu bài thi cũng có
  phần thông tin — đọc kỹ: có người fail task chỉ vì không đọc đề.
- Được dùng **Kubernetes documentation** tại kubernetes.io/docs — hãy
  dùng, đặc biệt để copy YAML code vào file giúp việc tạo resources dễ
  hơn.

## Phần 2 — Preparing the sample exam environment

- Sample exam này cần một **cài đặt Minikube hoàn toàn mới**. Một số
  task trong exam **yêu cầu Minikube** — không dùng Minikube thì **3
  task sẽ không chạy đúng** → khả năng cao fail sample exam.
- Đặt **timer 2 giờ** (nên báo thức trên điện thoại) — sample exam được
  làm representative với đề thật: hoàn thành được toàn bộ tasks trong 2
  giờ.
- Làm xong hết tasks → dùng script **examgrade.sh** trong course Git repo
  (github.com/sandervanvugt/ckad) để chấm, sau đó đối chiếu solutions.

### Demo — reset Minikube sạch

```bash
minikube status                 # Minikube đang chạy
minikube stop                   # dừng
minikube delete                 # xóa hẳn môi trường
minikube start --cni=calico     # khởi động lại với CNI calico
minikube status                 # verify đã chạy
kubectl get all                 # môi trường sạch
```

- `--cni=calico` thực sự cần cho **NetworkPolicy**.
- Điểm tốt: lần start sau khi delete **nhanh hơn lần đầu tiên** —
  container images đã có sẵn trên hệ thống nên không mất thời gian
  download lại.
- `kubectl get all` trả môi trường sạch — tại điểm này là sẵn sàng.

## Phần 3 — Working through the 15 sample exam tasks

Cách học đề xuất: theo dõi video đọc task → **dừng video** → tự làm
task → chuyển task kế. Làm xong **toàn bộ 15 task trong 2 giờ** rồi mới
chấm điểm và đối chiếu solutions.

### Task 1 — Namespace, pod và Secret

- Namespace **indiana**; trong namespace này tạo pod **inpod** chạy
  image `nginx:latest`.
- Cùng namespace: secret **insecret** lưu biến **color** với giá trị
  **blue**; pod phải dùng secret này để định nghĩa biến môi trường.

### Task 2 — Tìm pods theo label

- Tìm mọi pods có label `tier=control-plane`; ghi **danh sách tên** các
  pod đó — và không có gì khác — vào file **/tmp/task2pods**.

### Task 3 — ConfigMap mount vào pod

- Tạo file **index.html** với nội dung `Welcome to the task3 web
  server`.
- Tạo configmap **task3cm** chứa nội dung file này; chạy pod
  **oregonpod** với `nginx:latest`, mount configMap vào thư mục
  **/usr/share/nginx/html**.

### Task 4 — Sidecar

- Pod **sidepod**: chạy `nginx:latest` cùng một **sidecar container**
  chạy command `sleep 15`.

### Task 5 — Liveness probe

- Namespace **probes**; trong đó pod **probepod** chạy image busybox
  với command `sleep infinity`.
- Pod phải dùng **liveness probe** check **healthz endpoint trên
  kubernetes service** *(tương tự end-of-lesson lab Bài 17)*.

### Task 6 — Deployment và rolling update

- Deployment **updates**, image `nginx:1.17`, **3 replicas**, label
  `type=prod`.
- Khi update deployment phải dùng **rolling update** với **maxSurge
  50%** và **maxUnavailable 50%**; update deployment lên `nginx:latest`,
  sau đó **roll back** update này.

### Task 7 — Service và Ingress

- Dùng lệnh `minikube ip` lấy IP của minikube instance; thêm dòng sau
  vào **/etc/hosts** (đổi IP nếu cần cho đúng IP minikube của mình):

```bash
192.168.49.2 myapp.info
```

- Dùng ứng dụng **updates** đã tạo ở Task 6: tạo service **task7svc**;
  dùng **ingress** để ứng dụng truy cập được qua URL **myapp.info** —
  test bằng `curl myapp.info`.

### Task 8 — NetworkPolicy

- Pod **nevaginx** chạy `nginx:latest` với label `type=webapp`.
- Tạo NetworkPolicy đảm bảo **chỉ** applications có label `type=tester`
  mới có access tới pod này.
- Chạy pod **nevatest** với `busybox:latest`; từ pod này phải truy cập
  được pod nevaginx — test bằng command:

```bash
wget --spider --timeout=1 nevaginx
```

### Task 9 — hostPath volume

- `minikube ssh` mở session tới minikube host:

```bash
sudo mkdir /webapp
sudo sh -c 'echo Welcome to the store > /webapp/index.html'
exit
```

- Pod **storepod** chạy `nginx:latest`, dùng **hostPath volume** mount
  thư mục **/webapp** trên minikube host vào thư mục
  **/usr/share/nginx/html** trong pod.
- Pod phải accessible qua minikube host **port 32032**; test bằng curl
  tới minikube IP port 32032 — phải thấy text `Welcome to the store`.

### Task 10 — Helm

- Dùng Helm install chart **mysql** của Bitnami từ repository
  `https://charts.bitnami.com/bitnami`.

### Task 11 — Memory requests và limits

- Namespace **nebraska**; trong đó deployment **snowdeploy** chạy
  `nginx:latest`.
- Containers của deployment phải **request memory ban đầu 64Mi** và
  **giới hạn memory tối đa 128Mi**.

### Task 12 — Canary deployment

- Namespace **birds** — mọi resource của task này tạo trong namespace
  đó.
- Deployment **oldbird** chạy `nginx:1.17`; deployment **newbird** chạy
  `nginx:latest`.
- Expose cả hai deployment qua service **oldbirds**, nối tới các pods
  bằng selector label `type=oldbirds`; service accessible trên minikube
  port **32323**.
- Configure service **load balance 20% requests tới newbird** và **80%
  requests tới oldbird**.

### Task 13 — Security context

- Pod **securepod** chạy `nginx:latest`, áp các security restrictions:
  - Containers trong pod **không được phép escalate privileges**.
  - Containers trong pod được start với **supplemental group GID
    2000**.

### Task 14 — Dockerfile và export image

- Dùng Dockerfile có sẵn trong course Git repo để build image
  **myapp:1.0**; export image ra tar file **/tmp/myapp.tar**, đảm bảo
  tuân thủ **OCI**.

### Task 15 — ServiceAccount

- Namespace **oklahoma**; trong đó tạo service account **secure**.
- Pod **securepod** (trong namespace oklahoma) chạy `nginx:latest`,
  phải dùng service account **secure**.

## Phần 4 — Grading with examgrade.sh

Làm xong toàn bộ 15 task → trong course Git repo, chạy:

```bash
./examgrade.sh
```

- Script lần lượt kiểm tra từng task và báo biết task nào pass, task
  nào fail.
- Kết quả của chính instructor: tổng điểm **320**, ông đạt **280** —
  vẫn mắc một vài lỗi. Xem các video solutions để biết chính xác sai ở
  đâu.

## Phần 5 — Solutions

Thói quen chung của instructor khi làm đề:

- **Lập plan cho từng task trước khi gõ lệnh** — quyết định hướng làm
  (imperative, YAML copy từ docs, hay kết hợp) rồi mới bắt tay vào.
- Đặt tên file theo task (`task1.yaml`, `task3.yaml`…) — nếu cuối giờ
  còn thời gian quay lại kiểm tra thì dễ tìm.
- Copy syntax từ `--help` và copy YAML từ kubernetes.io/docs thay vì tự
  gõ — giảm thiểu lỗi.
- Chú ý **indentation** mỗi lần dán YAML từ docs vào file.
- **Kick out everything you have not been asked to do** — script chấm
  có thể kiểm tra cả những field thừa để lại từ ví dụ docs.

### Task 1 — Namespace, pod và Secret

Điểm khó: pod phải nhận secret qua biến môi trường, nhưng `kubectl set
env` **không chạy được trên pod** — chỉ chạy trên deployment. Giải pháp
của instructor: tạo một deployment trung gian, "mượn" block env của nó.

```bash
kubectl create ns indiana
kubectl create deploy task1 --image=nginx -n indiana
kubectl create secret generic insecret --from-literal=color=blue \
  -n indiana
kubectl set env --help | less     # xem syntax trước khi gõ
kubectl -n indiana set env --from=secret/insecret deploy/task1
kubectl run inpod --image=nginx:latest -n indiana \
  --dry-run=client -o yaml > task1.yaml
kubectl edit deploy -n indiana task1   # copy block env vừa set
```

- Dán block env vào phần container của `task1.yaml` — chú ý
  indentation; `kubectl apply -f task1.yaml` tạo pod. Deployment task1
  chỉ là bước trung gian.
- Lần đầu tạo deployment quên `-n indiana` → phải delete làm lại.
  Working with namespaces: quên `-n` là lỗi dễ mắc; có thể đổi current
  namespace bằng `kubectl config` nhưng sau đó phải nhớ đổi lại —
  instructor thích dùng `-n` trên từng lệnh.
- Verify: `kubectl get pods -n indiana -o yaml` — thấy environment
  được set từ secret.

### Task 2 — Tìm pods theo label

```bash
kubectl get pods -A --selector=tier=control-plane \
  | awk '{print $2}' > /tmp/task2pods
```

- Chạy awk ra màn hình **trước** khi redirect để confirm đúng; dòng
  đầu (header) còn thừa → sửa file bỏ dòng đó.
- Không tự tin với awk → ghi thẳng output vào file rồi dọn thủ công:
  đừng mất quá nhiều thời gian với command mình không chắc.

### Task 3 — ConfigMap mount vào pod

Không có cách imperative dễ để mount configMap vào pod → dùng docs.

```bash
echo Welcome to the task3 web server > index.html
kubectl create cm task3cm --from-file=index.html
kubectl get cm task3cm -o yaml    # verify key index.html + nội dung
```

- Docs: **Configure a Pod to use a ConfigMap** → lướt qua các ví dụ
  env (không cần) → mục **Add ConfigMap data to a volume** → copy pod
  YAML ví dụ vào `task3.yaml`.
- Sửa theo đề: name `oregonpod`; image `nginx:latest`; mountPath
  `/usr/share/nginx/html`; configMap name `task3cm`. Tên volume nội bộ
  giữ nguyên cũng được — miễn là nhất quán giữa volumeMounts và
  volumes.
- **Xóa `restartPolicy: Never`** — "in your dreams, nobody is asking
  for that": bỏ mọi thứ đề không yêu cầu, script chấm có thể soi ngay
  những field thừa đó.
- Verify:

```bash
kubectl exec -it oregonpod -- cat /usr/share/nginx/html/index.html
# Welcome to the task3 web server
```

### Task 4 — Sidecar

- Docs: **Sidecar containers** (tính năng beta từ Kubernetes 1.29) →
  copy ví dụ YAML (dạng Deployment) vào `task4.yaml`.
- Biến Deployment thành Pod: bỏ labels, bỏ phần spec của deployment
  (replicas/selector/template), giữ lại `containers` +
  `initContainers`; bỏ volumes và volumeMounts.
- Định nghĩa ở đây: **sidecar = init container có `restartPolicy:
  always`**.
- Container chính: `nginx:latest` (bỏ command của ví dụ). Sidecar: đề
  không chỉ định image — instructor chọn `alpine` vì chỉ cần có lệnh
  `sleep`; command `sleep 15`.
- Lỗi vấp phải: quên đổi `apiVersion` từ `apps/v1` về `v1` → lỗi "no
  matches for kind pod in version apps/v1". Sửa rồi apply.
- Verify: `kubectl get pods` → sidepod **2/2**; sidecar cứ mỗi 15 giây
  lại restart — đó là cách sidecar container kiểu này hoạt động.

### Task 5 — Liveness probe

- Instructor dùng lại file mẫu **healthz-probe.yaml** *(nhắc lại lab
  cuối Bài 17)*: pod `readiness-exec`, image busybox, command
  `sleep 3600`, probe exec `wget --no-check-certificate --spider` tới
  healthz endpoint trên kubernetes service.

```bash
kubectl apply -f healthz-probe.yaml
kubectl describe pod readiness-exec
```

- Pod Running; check lại sau vài giây thấy probe successful.
- *(File mẫu dùng pod `readiness-exec`, `sleep 3600` và readiness
  probe — đề bài yêu cầu pod `probepod`, `sleep infinity` và
  **liveness** probe; trên đề thi đặt đúng theo đề bài.)*

### Task 6 — Deployment và rolling update

```bash
kubectl create deploy updates --image=nginx:1.17 --replicas=3 \
  --dry-run=client -o yaml > task6.yaml
```

- Sửa task6.yaml: thêm label `type: prod` vào metadata; docs
  Deployment — search từ khóa "**maxSurge**" (đủ cụ thể để ra đúng
  phần cần) — copy block strategy:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 50%
    maxSurge: 50%
```

- Chú ý: `rollingUpdate` phải cùng level với `type` — lệch
  indentation là fail. Hai tham số này nhận cả số tuyệt đối lẫn phần
  trăm.
- Update rồi roll back:

```bash
kubectl apply -f task6.yaml
kubectl set image deployment/updates nginx=nginx:latest
kubectl get all --selector=app=updates
kubectl rollout status deploy/updates
kubectl rollout undo deploy/updates
```

### Task 7 — Service và Ingress

```bash
sudo vim /etc/hosts            # thêm <minikubeip> myapp.info
kubectl expose deploy updates --name=task7svc --port=80
kubectl get all --selector=app=updates   # thấy service đã tạo
```

- Ingress cần addon (reset Minikube làm mất) — kiểm tra và bật:

```bash
minikube addons list
minikube addons enable ingress
```

- Tạo ingress bằng lệnh imperative (xem ví dụ trong
  `kubectl create ing --help`), rồi test:

```bash
kubectl create ing myapp --rule=myapp.info/=task7svc:80
curl myapp.info
# Welcome to nginx!
```

### Task 8 — NetworkPolicy

```bash
kubectl run nevaginx --image=nginx:latest
kubectl label pod nevaginx type=webapp
kubectl expose pod nevaginx --port=80   # không gọi thẳng vào pod được
```

- NetworkPolicy thật sự nên đi bằng docs: trang **Network Policy** có
  ví dụ — copy vào `task8.yaml` rồi dọn: giữ `podSelector` với
  `type=webapp`; trong `ingress.from` bỏ `ipBlock`, bỏ
  `namespaceSelector`, giữ `podSelector` đặt `type=tester`; bỏ
  `policyTypes` và phần ports.
- Các lỗi instructor gặp lần lượt: (1) test ngay khi pod chưa expose →
  bad address; (2) expose xong vẫn timeout vì pod nguồn chưa có label
  `type=tester` — NetworkPolicy chặn tới khi label đúng:

```bash
kubectl apply -f task8.yaml
kubectl run nevatest --image=busybox -- sleep infinity
kubectl label pod nevatest type=tester
kubectl exec -it nevatest -- wget --spider --timeout=1 nevaginx
```

- Nếu làm đúng mọi thứ vẫn không chạy: phải dùng **Calico network
  plugin** — `minikube stop` → `minikube delete` → `minikube start
  --cni=calico`.

### Task 9 — hostPath volume

```bash
minikube ssh
sudo mkdir /webapp
sudo sh -c 'echo Welcome to the store > /webapp/index.html'
cat /webapp/index.html          # verify trên host
exit
```

- Đọc kỹ đề: **không ai yêu cầu persistent storage** → đừng phức tạp
  hóa với PV/PVC, dùng thẳng hostPath.
- Docs **Volumes** → ví dụ hostPath → copy vào `task9.yaml`, sửa: name
  `storepod`; image `nginx:latest`; bỏ 3 dòng os/nodeSelector;
  mountPath `/usr/share/nginx/html`; bỏ `readOnly: true`; hostPath
  `path: /webapp` (bỏ dòng type cuối).
- Lỗi nhỏ: gõ image có space (`nginx latest`) → lỗi line 8; sửa rồi
  apply lại.
- Expose qua NodePort 32032:

```bash
kubectl expose pod storepod --port=80     # fail: pod has no labels
kubectl label pod storepod type=storage   # label gì cũng được
kubectl expose pod storepod --port=80
kubectl edit svc storepod                 # type: NodePort + nodePort: 32032
curl 192.168.49.2:32032
# Welcome to the store
```

- Chi tiết dễ nhầm: key `nodePort` — n thường; value `NodePort` — N
  hoa.

### Task 10 — Helm

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install bitnami/mysql --generate-name
```

- Trên exam helm đã pre-installed; reset Minikube không ảnh hưởnghelm
  trên VM.
- Repo từng add từ session trước → "already exists… skipping", càng
  dễ.
- Verify: `kubectl get all` → MySQL chạy dạng **StatefulSet**; đang
  "work in progress" thì cứ để tự hoàn tất.

### Task 11 — Memory requests và limits

```bash
kubectl create ns nebraska
kubectl create deploy snowdeploy -n nebraska --image=nginx
kubectl set resources --help | less      # copy dòng ví dụ ra dùng
kubectl set resources -n nebraska deploy/snowdeploy \
  --limits=memory=128Mi --requests=memory=64Mi
kubectl describe -n nebraska deploy/snowdeploy | less
```

- Không gõ `:latest` thì mặc định đã lấy latest.
- Verify trong describe: thấy limits 128Mi và requests 64Mi.

### Task 12 — Canary deployment

```bash
kubectl create ns birds
kubectl -n birds create deploy oldbird --image=nginx:1.17 \
  --dry-run=client -o yaml > oldbird.yaml
kubectl -n birds create deploy newbird --image=nginx:latest \
  --dry-run=client -o yaml > newbird.yaml
```

- Sửa cả hai file: thêm label `type: oldbirds` ở **cả deployment lẫn
  pod template** — service select theo label này nên phải có xuống tới
  pod.
- *(Transcript phần đề bài đọc "all birds" — solution gõ "oldbirds"
  xuyên suốt; chuẩn hóa theo solution.)*

```bash
kubectl apply -f oldbird.yaml
kubectl apply -f newbird.yaml
kubectl -n birds expose deploy oldbird --name=oldbirds \
  --selector=type=oldbirds --type=NodePort --port=80
kubectl -n birds edit svc oldbirds      # nodePort: 32323
```

- Lần đầu expose quên `-n birds` → không thấy deployment. Với
  namespace luôn nhớ `-n`.
- Tỉ lệ 80/20 làm bằng số replicas — service load balance đều giữa các
  pod:

```bash
kubectl -n birds scale deploy/oldbird --replicas=4
kubectl -n birds describe svc oldbirds
```

- 4 pods oldbird + 1 pod newbird = 80% / 20%. Describe cho thấy service
  trỏ tới nhiều backing pods.

### Task 14 — Dockerfile và export image

- Dockerfile có sẵn trong course Git repo — tìm nó:

```bash
find . -name dockerfile
```

- Task này không phải về Docker — dùng Dockerfile nào cũng được, mục
  đích là **create image**:

```bash
docker build -t myapp:1.0 .
```

- "Ensuring it is OCI compliant": khỏi lo — **docker image mặc định đã
  OCI compliant**, chỉ việc `docker save`:

```bash
docker save myapp:1.0 > /tmp/myapp.tar
file /tmp/myapp.tar     # xác nhận đúng là tar ball
```

- `docker save -o /tmp/myapp.tar myapp:1.0` viết ra file thay vì
  standard out — instructor quên `-o` và dùng shell redirection `>`
  thay thế; cách nào cũng được, cái cần là kết quả.

### Task 15 — ServiceAccount

```bash
kubectl create ns oklahoma
kubectl run securepod -n oklahoma --image=nginx \
  --dry-run=client -o yaml > task15.yaml
```

- (Lần gõ đầu quên `--image`, phải thêm vào.)
- Không có command-line option để set serviceAccount cho pod → sửa
  `task15.yaml`: thêm `serviceAccount: secure` vào pod spec.
- Lỗi instructor gặp: gõ nhầm `secureaccount` → apply fail với lỗi
  "unknown field spec.secureaccount" — sửa lại đúng
  `serviceAccount`.
- Tạo service account trước rồi apply:

```bash
kubectl create -n oklahoma sa secure
kubectl apply -f task15.yaml
kubectl get pods -n oklahoma -o yaml | less   # serviceAccount: secure
```
