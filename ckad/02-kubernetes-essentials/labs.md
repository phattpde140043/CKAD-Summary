# Labs — Module 2: Kubernetes Essentials

Tổng hợp toàn bộ ví dụ demo và lab exercise từ các bài học trong module, chuyển thành checklist
thực hành kèm tiêu chí verify. Mục tiêu: làm tới khi không cần nhìn tài liệu.

## Bài 5 — Understanding and Using Pods

### Lab 5.1 — Chạy standalone pod đầu tiên (demo Phần 2)

- [ ] `kubectl get pods` — **Verify:** thấy pod do deployment `firstapp` tạo ở Bài 4 đang chạy
- [ ] `kubectl run firstpod --image=nginx` — `--image` là argument bắt buộc
- [ ] `kubectl get pods` — **Verify:** pod mới chuyển sang **Running**
- [ ] `kubectl describe pod firstpod | less` — output dài, pipe qua pager `less`; **Verify:** thấy đủ — **IP address** (pod giữ IP) · container + state `Running` · mục **Events** ở cuối (tổng kết mọi việc Kubernetes đã làm để chạy pod)

### Lab 5.2 — Khám phá properties bằng kubectl explain (demo Phần 2 & 3)

- [ ] `kubectl explain pod` — **Verify:** thấy top-level fields: `apiVersion`, `kind`, `metadata`, `spec`, `status`
- [ ] `kubectl explain pod.spec | less` — **Verify:** danh sách rất dài các properties (`activeDeadlineSeconds`, `automountServiceAccountToken`, `dnsConfig`...) kèm giải thích từng cái
- [ ] `kubectl explain pod.spec.containers | less` — **Verify:** thấy các property của container: `name`, `image`, `command`, `args`, `env`

### Lab 5.3 — Generate YAML rồi apply (demo Phần 4)

- [ ] `kubectl run mynginx --image=nginx --dry-run=client -o yaml` — chạy **không** redirect trước; **Verify:** YAML in ra màn hình, cluster chưa tạo resource gì
- [ ] `kubectl run mynginx --image=nginx --dry-run=client -o yaml > mynginx.yaml` — ghi YAML vào file
- [ ] `kubectl apply -f mynginx.yaml` — **Verify:** `kubectl get pods` thấy pod `mynginx` **Running**
- [ ] Ghi nhớ thi: đừng viết YAML tay — generate bằng `--dry-run=client -o yaml > file.yaml` hoặc copy-paste từ kubernetes.io/docs

### Lab 5.4 — Multi-container pod (demo Phần 5)

- [ ] Mở file `multi-container.yaml` trong course repo (đã clone ở Lab 1.0) — **Verify:** pod `multi-container` với `containers` là **list** 2 mục: busybox (`command: sleep 3600`) + nginx
- [ ] `kubectl apply -f multi-container.yaml`
- [ ] `kubectl get pods multi-container` — **Verify:** cột READY ban đầu **0/2** (2 containers kỳ vọng, 0 ready); chờ vài giây get lại → **2/2**

### Lab 5.5 — Namespace và chạy pod trong namespace (demo Phần 6)

- [ ] `kubectl get pods -A` — **Verify:** cột đầu là namespace name, liệt kê pod ở mọi namespace
- [ ] `kubectl create ns secret` — namespace phải tồn tại trước khi chạy resource trong nó
- [ ] `kubectl run secret --image=nginx -n secret` — `-n` chỉ định namespace
- [ ] `kubectl get pods` — **Verify:** KHÔNG thấy pod (get không có `-n` chỉ hiện namespace `default`)
- [ ] `kubectl get pods -n secret` — **Verify:** pod `secret` **Running**
- [ ] Ghi nhớ từ sai lầm trong demo: `kubectl run pod secretpod ...` là SAI — `kubectl run` chỉ dành cho pod, không ghi resource type ("pod" thành tên pod, "secretpod" thành argument → CrashLoopBackOff; exit code 127); dọn bằng `kubectl delete -n secret pod pod` (`pod` hai lần: resource type + tên)

### Lab 5.6 — Troubleshooting pod fail (demo Phần 7)

- [ ] `kubectl run mydb --image=mariadb` — trong lúc chờ (~30s): `kubectl get pods` → **ContainerCreating**; `kubectl describe pod mydb` → Events: scheduler gán node trước, node pull image sau
- [ ] `kubectl get pods` — **Verify:** STATUS lỗi + **RESTARTS** tăng (vd 3) — default restart policy start lại ứng dụng fail
- [ ] `kubectl describe pod mydb` — **Verify:** event cuối *Back-off restarting failed container*; container State **Waiting** / Reason **CrashLoopBackOff**, **Exit Code 1** (0 = success, khác 0 = fail)
- [ ] `kubectl logs mydb` — **Verify:** error của ứng dụng: MariaDB đòi `MARIADB_ROOT_PASSWORD`
- [ ] `kubectl delete pod mydb` → `kubectl run mydb --image=mariadb --env MARIADB_ROOT_PASSWORD=password` — **Verify:** pod `mydb` **Running**
- [ ] Ghi nhớ: `kubectl exec -it` chỉ dùng được khi pod đang chạy; set password qua `--env` chỉ để demo — production dùng ConfigMap/Secret (Module 5)

### Lab 5.7 — End-of-lesson lab Bài 5 (namespace + pod declarative)

Đề bài: một YAML file tạo namespace `microservice` + pod `microdb` (MariaDB) chạy trong namespace đó, tạo resources theo kiểu declarative.

- [ ] `kubectl create ns microservice --dry-run=client -o yaml > lab5.yaml` — `>` tạo mới/overwrite
- [ ] `kubectl run microdb --image=mariadb --env MARIADB_ROOT_PASSWORD=password -n microservice --dry-run=client -o yaml >> lab5.yaml` — `>>` append vào cuối file
- [ ] Sửa `lab5.yaml`: thêm dòng `---` trên đầu **mỗi** resource (namespace + pod) — separator đánh dấu YAML document mới
- [ ] `kubectl apply -f lab5.yaml` — **Verify:** cả namespace `microservice` lẫn pod `microdb` được tạo
- [ ] `kubectl get pods -n microservice` — **Verify:** `microdb` **Running**

## Bài 6 — Advanced Pod Features

### Lab 6.1 — Init container (demo Phần 1)

- [ ] Trên kubernetes.io tìm "init container" → mở mục **Configure Pod Initialization** (kỳ thi: dùng kết quả tìm kiếm thứ 1 hoặc 2) — **Verify:** thấy example "create a pod that has an init container"
- [ ] Copy example vào `init-demo.yaml` — **Verify:** hiểu cấu trúc — main container nginx port 80 · volume `workdir` (`emptyDir`) mount tại `/usr/share/nginx/html` · init container busybox `wget -O` ghi `index.html` từ info.cern.ch vào `workdir`
- [ ] `kubectl apply -f init-demo.yaml`
- [ ] `kubectl get pods` — **Verify:** STATUS `Init:0/1` khi init container đang chạy; get lại sau → **Running**
- [ ] `kubectl exec -it init-demo -- cat /usr/share/nginx/html/index.html` — **Verify:** thấy nội dung trang CERN — init container đã chép content vào volume chia sẻ với nginx

### Lab 6.2 — Sidecar container (demo Phần 2)

- [ ] Trên kubernetes.io mở mục **Sidecar containers** (KHÔNG dùng ví dụ trên discuss.kubernetes.io — site không truy cập được trên kỳ thi)
- [ ] Lấy example (dạng Deployment) → tách phần dưới `template` thành pod riêng `sidecar-pod.yaml`: `apiVersion: v1` · `kind: Pod` · metadata name `sidecar-pod` · `spec` — **Verify:** đủ 4 thành phần, đặc biệt `kind` (dễ quên khi copy tay)
- [ ] **Verify:** pod có container `myapp` (ghi log vào `/opt/logs.txt`) + `initContainers` `log-shipper` (alpine, `tail -f /opt/logs.txt`) kèm **`restartPolicy: Always`** — thứ biến init container thành sidecar (K8s ≥ 1.29)
- [ ] Có thể đơn giản hóa command sidecar thành `sleep 15` như demo
- [ ] `kubectl apply -f sidecar-pod.yaml`
- [ ] `kubectl get pods` — **Verify:** `Init:0/1` → Running; READY **0/2 → 2/2** (sidecar được đếm là container thật, khác init thường); **RESTARTS tăng liên tục** (sleep 15 xong → restart theo Always)

### Lab 6.3 — Port forwarding (demo Phần 3)

- [ ] `kubectl get pods -o wide` — **Verify:** thấy cột IP của từng pod
- [ ] `curl <pod-ip>` — **Verify:** không phản hồi — pod IP nằm trên mạng isolated
- [ ] `kubectl run fw-nginx --image=nginx`
- [ ] `kubectl port-forward fw-nginx 8080:80 &` — `&` chạy nền
- [ ] `curl localhost:8080` — **Verify:** nhận "Welcome to nginx!" — port 8080 trên máy chạy kubectl forward vào port 80 của pod
- [ ] Ghi nhớ: port forwarding chỉ để **test** — expose cho user dùng services/ingress (Module 4)

### Lab 6.4 — Restart policy (demo Phần 4)

- [ ] `kubectl run nginx1 --image=nginx`
- [ ] `kubectl get pods nginx1 -o yaml | grep restartP` — **Verify:** `restartPolicy: Always`; ghi nhớ `-o yaml` in toàn bộ current state của **mọi** resource
- [ ] `kubectl delete pods nginx1` — **Verify:** pod đã xóa vĩnh viễn — restartPolicy **không** restart pod bị delete
- [ ] `kubectl run nginx2 --image=nginx`
- [ ] `minikube ssh` → `crictl ps | grep nginx2` — **Verify:** thấy container nginx2, cột đầu là container ID (CRI = Container Runtime Interface; `crictl` dùng như docker/podman CLI)
- [ ] `crictl stop <container-id>` → `exit` → `kubectl get pods nginx2` — **Verify:** RESTARTS **1 (15s ago)** — container restart nhờ restartPolicy Always
- [ ] Ghi nhớ: restartPolicy Always chỉ restart **container**; muốn pod được thay thế khi fail → Deployment

### Lab 6.5 — Jobs (demo Phần 5)

- [ ] `kubectl create job -h` — **Verify:** ready-to-use examples nằm ở **đầu** output (mẹo thi)
- [ ] `kubectl create job onejob --image=busybox -- date`
- [ ] `kubectl get jobs,pods` — **Verify:** job `onejob` **Complete**; pod `onejob-<suffix>` với suffix tự sinh
- [ ] `kubectl get pods <onejob-pod> -o yaml | grep restartP` — **Verify:** `restartPolicy: Never`
- [ ] `kubectl delete job onejob`
- [ ] `kubectl create job mynewjob --image=busybox --dry-run=client -o yaml -- sleep 5 > mynewjob.yaml`
- [ ] Sửa `mynewjob.yaml`: trong **job spec** (top-level `spec`, KHÔNG phải pod spec dưới `template`) thêm `completions: 3` + `ttlSecondsAfterFinished: 60`
- [ ] `kubectl apply -f mynewjob.yaml` → `kubectl get jobs,pods` — **Verify:** completions đi dần 0/3 → 3/3
- [ ] Chờ ~60 giây → `kubectl get jobs,pods` — **Verify:** job + pods **tự động bị xóa** (TTL)
- [ ] Ghi nhớ: dùng job thì luôn set `ttlSecondsAfterFinished` — không có nó không ai dọn job giúp

### Lab 6.6 — Cron Jobs (demo Phần 6)

- [ ] `kubectl create cronjob -h | less` — **Verify:** examples ở đầu output; example **thứ hai** (có `-- date`) mới dùng được — example đầu chạy busybox không kèm command, vô nghĩa
- [ ] `kubectl create cronjob runme --image=busybox --schedule="*/2 * * * *" -- echo greetings from the cluster` — đếm đủ **5 vị trí**: minute · hour · day of month · month · day of week; `*/1` là redundant (`*` đã nghĩa mỗi phút)
- [ ] `kubectl get cronjobs,jobs,pods` — **Verify:** cronjob `runme` (LAST SCHEDULE vài giây) → job `runme-<suffix>` → pod đang chạy — đủ chuỗi cronjob → job → pod
- [ ] `kubectl create job runme --from=cronjob/runme` — **Verify:** job chạy **ngay** không phải chờ lịch (cách test cron job trên kỳ thi, nhất là cron job chạy 1 lần/ngày)
- [ ] `kubectl logs <runme-pod>` — **Verify:** thấy `greetings from the cluster` — pod không nối stdout, chỉ `kubectl logs` cho biết ứng dụng đã làm gì
- [ ] `kubectl delete cronjob runme` → `kubectl get cronjobs,jobs,pods` — **Verify:** sạch cả chuỗi — xóa top-level thì jobs + pods theo sau cũng biến mất
- [ ] Ghi nhớ: xóa resource **top-level** (cronjob/deployment); xóa lower-level thì nó được tạo lại

### Lab 6.7 — Clean up resources (demo Phần 7)

- [ ] `kubectl delete all` — **Verify:** không xóa gì — lệnh thiếu tên resource
- [ ] `kubectl delete all --all` — **Verify:** xóa pods/services/deployments của namespace hiện tại; service `kubernetes` bị xóa nhưng **tự tạo lại** (essential service)
- [ ] Ghi nhớ: pod do deployment quản lý → xóa **deployment**, không xóa pod (xóa pod sẽ được tạo lại)
- [ ] Ghi nhớ: KHÔNG BAO GIỜ `--force --grace-period=-1` (resource kẹt state deleting → fail phần thi đó); KHÔNG BAO GIỜ thêm `-A` vào delete all (xóa luôn system resources) — trong demo tác giả gõ lệnh rồi **Ctrl+U** hủy

### Lab 6.8 — End-of-lesson lab Bài 6 (cron job + activeDeadlineSeconds)

Đề bài: cron job chạy `sleep 30`; nếu sau 15 giây vẫn chạy → phải bị kill, chu kỳ bắt đầu lại.

- [ ] `kubectl explain cronjob.spec | less` — **Verify:** chỉ có timing properties (`startingDeadlineSeconds` = deadline **start** job nếu lỡ lịch — không phải cái cần)
- [ ] `kubectl explain cronjob.spec.jobTemplate.spec | less` — **Verify:** thấy `activeDeadlineSeconds` — job bị terminate nếu chạy liên tục quá số giây này → đúng yêu cầu đề
- [ ] `kubectl create cronjob sleepy --image=busybox --schedule="* * * * *" --dry-run=client -o yaml -- sleep 30 > lab6.yaml` — đề không chỉ định schedule → mỗi phút; cần YAML vì `activeDeadlineSeconds` chỉ thêm được qua file
- [ ] Sửa `lab6.yaml`: thêm `activeDeadlineSeconds: 15` vào **`spec.jobTemplate.spec`** (job spec — KHÔNG phải pod template bên dưới `template`)
- [ ] `kubectl apply -f lab6.yaml` → `kubectl get all` — **Verify:** cronjob `sleepy` xuất hiện; job chạy ở đầu phút nên chờ vài giây
- [ ] Get lại — **Verify:** pod chuyển **Terminating** (demo: pod ~19 giây tuổi) dù `sleep 30` chưa xong — activeDeadlineSeconds đã terminate job; phút kế tiếp chu kỳ lặp lại
- [ ] Ghi nhớ: trước khi làm lab, **phân tích yêu cầu → tìm property đúng mức spec** bằng `kubectl explain` (cronjob.spec ≠ jobTemplate.spec ≠ pod template)

## Bài 7 — Storage

### Lab 7.1 — Pod volume emptyDir (demo Phần 2)

- [ ] Trên kubernetes.io tìm "configure a volume for a pod" → copy example vào `podvolume.yaml` — **Verify:** pod Redis + volume `redis-storage` (emptyDir) + volumeMounts mount vào `/data/redis`; ghi nhớ: không có lệnh imperative tạo pod volume → dùng docs
- [ ] `kubectl apply -f podvolume.yaml`
- [ ] `kubectl describe pod redis | less` — **Verify:** Mounts `/data/redis` from `redis-storage` + Volumes Type **EmptyDir**
- [ ] `kubectl exec -it redis -- touch /data/redis/hello-world`
- [ ] `minikube ssh` → `crictl stop $(crictl ps | awk '/redis/ {print $1}')` — command substitution `$()` tự tìm container ID (không cần copy tay)
- [ ] `exit` → `kubectl get pods` — **Verify:** pod redis đã restart; `kubectl exec -it redis -- ls -l /data/redis` — **Verify:** `hello-world` **vẫn còn** — volume thuộc về pod, sống sót qua container restart
- [ ] `minikube ssh` → `sudo find / -name hello-world 2>/dev/null` — **Verify:** thấy file trong path kubelet dưới `/var/lib/kubelet/pods/<pod-ID>/volumes/` — không cần nhớ path chính xác, quan trọng là hiểu storage nằm trên host thế nào
- [ ] `kubectl delete pod redis` → minikube `find` lại — **Verify:** file **biến mất** — emptyDir bị xóa theo pod
- [ ] Ghi nhớ: emptyDir chỉ hợp để **lưu tạm** + **share files giữa containers trong multi-container pod**; ephemeral + host-bound → real life không phải giải pháp tốt

### Lab 7.2 — Persistent Volume thủ công (demo Phần 3)

- [ ] `kubectl explain persistentvolume.spec | less` — **Verify:** thấy toàn bộ storage types address trực tiếp: `awsElasticBlockStore`, `azureDisk`, `azureFile`... kèm mô tả
- [ ] Docs tìm "create a persistent volume" (example **khó tìm** — practice trước kỳ thi) → copy example vào `pvdemo.yaml` — **Verify:** đủ 5 properties: name `task-pv-volume` · storageClassName `manual` · capacity `10Gi` · accessModes `ReadWriteOnce` · hostPath `/mnt/data`
- [ ] Ghi nhớ 3 access modes: `ReadWriteOnce` (common & safe) · `ReadOnlyMany` · `ReadWriteMany`
- [ ] `kubectl apply -f pvdemo.yaml`
- [ ] `kubectl get pv` — **Verify:** PV `task-pv-volume` **Available** — 10Gi · RWO · storage class `manual`
- [ ] `kubectl describe pv task-pv-volume` — **Verify:** Type **HostPath**, Path `/mnt/data`
- [ ] Ghi nhớ: PV manual bắt buộc có `storageClassName` (PVC sẽ bind qua property này — cần ở cả 2 đầu); CKAD dùng hostPath là đủ, PV chưa dùng được cho tới khi có PVC

### Lab 7.3 — Storage class trên Minikube (demo Phần 4)

- [ ] `minikube addons list` — **Verify:** đa số add-on disabled (mỗi add-on bật thêm tiêu tốn resource máy), nhưng `storage-provisioner` **enabled** — Minikube mặc định đã có provisioner
- [ ] `kubectl get storageclass` — **Verify:** thấy storage class `standard`
- [ ] `kubectl describe storageclass standard` — **Verify:** provisioner **minikube-hostpath** — tạo hostPath storage on demand; "không spectacular" nhưng đủ để học
- [ ] Ghi nhớ: provisioner **không thuộc vanilla Kubernetes** (do ecosystem cung cấp, có thể tích hợp trong distribution); storage class không có provisioner chạy phía sau thì **không làm gì**; cấu hình SC + provisioner **không nằm trong đề CKAD**

### Lab 7.4 — PVC bind PV (demo Phần 5)

- [ ] Docs trang "create a persistent volume" → scroll tới **create a persistent volume claim** → copy vào `pvcdemo.yaml` — **Verify:** name `task-pv-claim` · storageClassName `manual` · accessModes `ReadWriteOnce` · resources **3Gi** (G hoa i thường = gibibytes)
- [ ] `kubectl apply -f pvcdemo.yaml` → `kubectl get pvc,pv` — **Verify:** PVC **Bound**; CAPACITY **10Gi** dù chỉ request 3Gi — bind **all or nothing**: PV offer 10 → PVC dùng trọn 10, không chia sẻ PV cho PVC khác
- [ ] Sửa `pvcdemo.yaml`: **xóa storageClassName** + đổi name → apply → `kubectl get pvc,pv` — **Verify:** PVC bind vào PV `pvc-<uuid>` storage class `standard` — PV do storage class Minikube **tạo tự động** (dynamic provisioning)
- [ ] Ghi nhớ: PVC không set storageClassName → chỉ bind PV **do storage class tạo** (SC là default provider); **không có SC available + không có storageClassName → không bind, storage dysfunctional**

### Lab 7.5 — Pod dùng PVC (demo Phần 6)

- [ ] Cùng trang docs → phần **create a pod** → copy vào `podpvc.yaml` — **Verify:** volume `task-pv-storage` type **persistentVolumeClaim** + `claimName: task-pv-claim` + mountPath `/usr/share/nginx/html`
- [ ] `kubectl apply -f podpvc.yaml` → `kubectl describe pod task-pv-pod` — **Verify:** Volumes `task-pv-storage` đến từ PVC — pod **chỉ biết claim name**, không biết storage type phía sau
- [ ] `kubectl exec task-pv-pod -- touch /usr/share/nginx/html/testfile`
- [ ] `kubectl describe pv task-pv-volume` — **Verify:** Type HostPath · Path `/mnt/data`
- [ ] `minikube ssh` → `ls /mnt/data` — **Verify:** thấy `testfile` — chuỗi: mount point pod → PVC → PV → directory host
- [ ] Ghi nhớ: khi chạy ứng dụng, đặt YAML pod + YAML PVC cạnh nhau (cùng file/repo); hostPath dễ tìm file — storage type khác không chắc

### Lab 7.6 — End-of-lesson lab Bài 7 (PV hostPath + pod busybox + kubectl cp)

Đề bài: PV hostPath directory `/storage` + access mode **ReadWriteMany**; pod busybox chạy `sleep infinity`, mount PV storage vào `/data`; `kubectl cp /etc/hosts` từ máy vào PV storage; verify trên storage host.

- [ ] PV: copy example từ docs vào `lab7pv.yaml` — **Verify:** name `lab7pv` · storageClassName `manual` · accessModes **ReadWriteMany** · hostPath **`/storage`** → `kubectl apply -f lab7pv.yaml` — *copy lại từ đầu, đừng tái dùng file cũ — trên kỳ thi không có sẵn file*
- [ ] PVC: copy vào `lab7pvc.yaml` — **Verify:** name `lab7pvc` · accessModes **ReadWriteMany** (khớp PV) → apply → `kubectl get pv,pvc` — **Verify:** lab7pv **Bound** to lab7pvc; lưu ý `default/lab7pvc` — `default` là **namespace**, không phải storage class
- [ ] Pod: copy vào `lab7pod.yaml` — **Verify:** name `lab7pod` · image **busybox** · volume giữ tên nội bộ `task-pv-storage` nhưng đổi `claimName: lab7pvc` · volumeMounts mountPath **`/data`** — cẩn thận: trong demo tác giả apply xong mới phát hiện mountPath còn `/usr/share/nginx/html` → phải `kubectl delete -f lab7pod.yaml` sửa lại
- [ ] Tìm syntax `sleep infinity`: `kubectl run dummy --image=busybox --dry-run=client -o yaml -- sleep infinity` — **Verify:** YAML cho thấy `args: ["sleep", "infinity"]` → thêm args vào pod
- [ ] `kubectl cp -h` — **Verify:** ghi chú **requires tar**; example copy `/tmp/foo` local → `/tmp/bar` trong remote pod
- [ ] `kubectl cp /etc/hosts lab7pod:/data`
- [ ] Verify cuối: `kubectl describe pv lab7pv` → hostPath `/storage`; `minikube ssh` → `ls -l /storage` — **Verify:** thấy file `hosts` — proof lab hoàn thành
- Ghi nhớ: access mode là **khai báo**, không đảm bảo năng lực thật của storage — hostPath không share được giữa nhiều node; RWX ở lab này chạy được vì Minikube chỉ có 1 node *(bổ sung từ nghiên cứu ngoài video)*

## Deep dive labs (bổ sung ngoài video)

### Lab DD.1 — Job failure & retry

- [ ] Tạo Job fail lập tức và quan sát retry:

```bash
kubectl create job failjob --image=busybox -- false
kubectl get jobs,pods -w
```

**Verify:** pod `failjob-<suffix>` Failed với exit code 1; sau ~10s,
20s, 40s... pod **mới** với suffix khác lần lượt xuất hiện và cũng
Failed — đó là backoff của backoffLimit (mặc định 6).

- [ ] Xem Job chịu thua:

```bash
kubectl describe job failjob
kubectl get pods -l job-name=failjob
```

**Verify:** Events hiện `BackoffLimitExceeded` — "Job has reached the
specified backoff limit"; tổng số pod Failed nhiều hơn 1 (lần đầu +
các retry).

- [ ] `activeDeadlineSeconds` chấm dứt Job treo:

```bash
kubectl create job slowjob --image=busybox --dry-run=client -o yaml \
  -- sleep 300 > slowjob.yaml
```

Sửa `slowjob.yaml`: thêm `activeDeadlineSeconds: 30` vào `spec` của
Job (top-level, cạnh `template`) — cùng vị trí như Lab 6.8 — rồi:

```bash
kubectl apply -f slowjob.yaml
kubectl get job slowjob
```

**Verify:** Job **FAILED** sau đúng ~30 giây; `kubectl describe job
slowjob` → Conditions: Failed True, Reason: `DeadlineExceeded`; pod bị
Terminating trước khi `sleep 300` chạy xong.

- [ ] Đóng băng CronJob:

```bash
kubectl create cronjob ticker --image=busybox \
  --schedule="* * * * *" -- echo tick
kubectl get jobs -w                     # Job mới mỗi phút
kubectl patch cronjob ticker -p '{"spec":{"suspend":true}}'
kubectl get cronjob ticker
```

**Verify:** cột SUSPEND = **true**; các phút sau đó không còn Job mới
sinh ra.

- [ ] Dọn dẹp:

```bash
kubectl delete job failjob slowjob
kubectl delete cronjob ticker
```

**Ghi nhớ:** pod fail ≠ Job fail ngay — backoffLimit cho Job nhiều cơ
hội thử lại; muốn Job chết đúng hẹn thì activeDeadlineSeconds, muốn
nghỉ ngơi thì suspend.
