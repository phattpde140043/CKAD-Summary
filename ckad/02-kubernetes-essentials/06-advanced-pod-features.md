# Bài 6 — Advanced Pod Features

Bài học về các tính năng nâng cao của pod, tiếp nối pod essentials ở Bài 5:
init containers, sidecar container, port forwarding, restart policy, jobs &
cron jobs, clean up resources.

## Lộ trình bài học

1. Init containers và sidecar container — sidecar có nghĩa đặc biệt từ Kubernetes version gần đây
2. Port forwarding — cách đơn giản kiểm tra pod đang làm gì
3. Restart policy — pod parameter quan trọng
4. Jobs và cron jobs
5. Clean up resources khi mọi thứ trục trặc
6. End-of-lesson lab

## Phần 1 — Init containers

### Khái niệm

- Init container là **special case của multi-container pod**.
- Init container **runs to completion** — chạy đến khi xong việc **trước khi
  main container được start**.
- Main container chỉ start khi init container **thành công** — init container
  fail thì main container **không bao giờ start**.

### Demo — lấy code từ Kubernetes documentation

Tác giả demo bằng cách dùng **chính documentation**, vì đây cũng là thứ dùng
trên kỳ thi. Tìm "init container" trên kubernetes.io → 2 kết quả: *Init
Containers* và *Configure Pod Initialization*.

> **Mẹo thi:** kết quả cần dùng luôn là **mục đầu tiên hoặc mục thứ hai** của
> kết quả tìm kiếm. Thậm chí không cần đọc kỹ — copy code mẫu về terminal rồi
> sửa.

Copy ví dụ "create a pod that has an init container" vào `init-demo.yaml`, đọc
line by line:

| Thành phần | Nội dung |
|---|---|
| `metadata.name` | `init-demo` |
| Container chính | `nginx`, port 80 |
| Volume | `workdir` kiểu `emptyDir` — môi trường tạm cho **shared storage**, mount tại `/usr/share/nginx/html` |
| Init container | image `busybox:1.28`, command `wget -O` ghi `index.html` vào `workdir`, nguồn `http://info.cern.ch` |

Luồng hoạt động:

1. Init container chạy `wget` — tải trang `info.cern.ch` *(fun fact: website
   đầu tiên trong lịch sử máy tính)* ghi thành `index.html` vào volume
   `workdir` (init container cũng mount volume này).
2. Main container (nginx) mount **cùng volume** `workdir` tại
   `/usr/share/nginx/html` → có sẵn content do init container tạo ra.

### Chạy và verify

```bash
kubectl apply -f init-demo.yaml
kubectl get pods    # STATUS: Init:0/1 — init container đang làm việc
kubectl get pods    # chờ một chút → Running
kubectl exec -it init-demo -- cat /usr/share/nginx/html/index.html
```

- **`Init:0/1`** là trạng thái thấy được khi init container còn đang chạy —
  bình thường tự mất.
- Lệnh `exec ... -- cat` hiện nội dung `index.html` lấy từ CERN → chứng minh
  init container đã chạy đúng việc của nó.

*(Transcript ghi "user share nginx.html" và "wget-o" — chuẩn hóa thành
`/usr/share/nginx/html` và `wget -O`.)*

## Phần 2 — Sidecar container (Kubernetes ≥ 1.29)

### Định nghĩa mới

- Định nghĩa sidecar **đã thay đổi từ Kubernetes v1.29** (ra mắt ở dạng beta;
  khóa ghi hình trên K8s 1.30) — thông tin cũ hơn là outdated.
- **Sidecar container = init container có `restartPolicy: Always`.**
  - Init container thường: làm một job **một lần rồi kết thúc**.
  - Sidecar: start **trước main container** nhưng **chạy liên tục** — xong
    một vòng lại restart; thường dùng để **repeatedly chạy một command**.
- **Không tồn tại attribute `sidecar`** — không có container property hay pod
  property nào tên sidecar. Chỉ là init container với restartPolicy Always.
- Giống init container thường: **phải hoàn thành ít nhất một lần** trước khi
  main pod start.

### Demo — từ Deployment example về standalone pod

- Trên docs tìm "sidecar": ví dụ "simple sidecar application" nằm trên
  discuss.kubernetes.io — **site không truy cập được trên kỳ thi**; dùng mục
  **Sidecar containers** trên kubernetes.io.
- Ví dụ trong docs là **Deployment** — chưa học (phần sau khóa); tạm hiểu:
  deployment là **wrapper quanh pod template**, mọi thứ dưới `template` là
  cấu hình pod thông thường.
- Bài học rút ra: tách pod spec ra file riêng `sidecar-pod.yaml` — `apiVersion:
  v1`, `kind: Pod`, `metadata` (name `sidecar-pod` + labels), `spec` thụt lề
  cùng mức với metadata. **Dễ quên nhất là `kind`** khi copy thủ công.

Nội dung pod:

- Container chính `myapp` — chạy command ghi log vào `/opt/logs.txt`.
- `initContainers`: `log-shipper`, image `alpine:latest`, command `sh -c
  "tail -f /opt/logs.txt"` — theo dõi và xuất nội dung log.
- Init container này là sidecar vì có **`restartPolicy: Always`**.
- Tác giả đơn giản hóa command sidecar thành `sleep 15` để làm rõ bản chất:
  sidecar vẫn là init container — chạy xong 15 giây rồi restart, lặp lại mãi.

### Chạy và quan sát

```bash
kubectl apply -f sidecar-pod.yaml
kubectl get pods   # Init:0/1 → Running; RESTARTS tăng dần
```

- READY **`0/2` ngay từ đầu** — khác init-demo (READY `1/1`): init container
  thường **không được đếm** là container, còn sidecar (restartPolicy Always)
  **được đếm** như container thật.
- Pod Running và **RESTARTS tăng liên tục** — `sleep 15` kết thúc rồi restart
  theo restartPolicy Always. Đó là cách sidecar hoạt động từ K8s 1.29.

*(Transcript ghi "tilt-f opt-logs.txt" và "alpine-latest" — chuẩn hóa thành
`tail -f /opt/logs.txt` và `alpine:latest`.)*

## Phần 3 — Port forwarding

### Khái niệm

- Cách **đơn giản để verify truy cập đến pod** — dành cho **testing**.
- Pod có thể được truy cập nhiều cách: services, ingress, Gateway API (học
  phần sau) — trước mắt dùng port forwarding.
- `kubectl port-forward` **expose port của pod trên máy đang chạy kubectl**
  (kubectl host = workstation của bạn) và forward về pod:

```bash
kubectl port-forward fw-nginx 8080:80
```

Expose port **8080 trên kubectl host** → forward đến **port 80 trong pod**.
Chỉ dùng để **test pod accessibility trên cluster node** — KHÔNG dùng để
expose ứng dụng cho user bên ngoài; user access đi qua services & ingress.

### Demo

```bash
kubectl get pods -o wide    # hiện IP của từng pod
curl 10.244.0.5             # KHÔNG trả lời — pod IP nằm trên isolated network
kubectl run fw-nginx --image=nginx
kubectl port-forward fw-nginx 8080:80 &   # & : chạy nền, tránh treo màn hình
curl localhost:8080         # "Welcome to nginx!"
```

- Pod IP nằm trên **mạng isolated** — `curl` thẳng đến IP không được trả lời,
  không truy cập được từ bên ngoài; đó là lý do cần port forwarding để test.
- `&` chạy port-forward ở background; nhấn Enter để lấy lại prompt.

*(Transcript ghi pod IP "10.2.44.0.5" — chuẩn hóa thành `10.244.0.5` theo dải
pod IP mặc định của Minikube.)*

## Phần 4 — Restart policy

### Khái niệm

- `restartPolicy` là **attribute trong pod specification** — quyết định điều
  gì xảy ra khi **container** do pod quản lý **crash**.
- Giá trị mặc định `Always` → container được **restart sau khi crash**.
- **Điểm dễ hiểu nhầm nhất:** `restartPolicy: Always` **không ảnh hưởng state
  của cả pod** — pod bị stopped/killed thì **không** được restart. Nghĩ "Always
  sẽ restart pod của tôi" là sai.
- Muốn pod được thay thế khi fail → đưa vào **Deployment** — đó là cách duy
  nhất.

### Demo 1 — restartPolicy không cứu pod bị delete

```bash
kubectl run nginx1 --image=nginx
kubectl get pods nginx1 -o yaml | grep restartP   # → restartPolicy: Always
kubectl delete pods nginx1                        # pod đã xóa, không hồi sinh
```

- `kubectl get <resource> -o yaml` — option **mới**, chưa gặp trước đây: in ra
  **toàn bộ current state** của bất kỳ resource nào trong Kubernetes.
- Output rất dài → pipe qua `grep`; `kubectl delete` xong là pod vẫn bị xóa —
  `restartPolicy: Always` không can thiệp.

### Demo 2 — chứng minh container được restart (behind the scenes)

Quan sát từ control node — với Minikube là SSH thẳng vào node:

```bash
kubectl run nginx2 --image=nginx
minikube ssh                     # vào Kubernetes control node
crictl ps | grep nginx2          # thấy container nginx2, cột đầu = container ID
crictl stop <container-id>       # stop container trực tiếp
exit
kubectl get pods nginx2          # RESTARTS: 1 (15s ago)
```

- **CRI = Container Runtime Interface** — phần của Kubernetes **chạy
  containers**; việc này KHÔNG do Docker hay Podman đảm nhiệm mà do CRI.
- `crictl` là utility của CRI — dùng giống `docker`/`podman` CLI; `crictl ps`
  liệt kê **mọi container đang chạy trên node** (rất nhiều, nên grep).
- `crictl stop` nhận **tên container hoặc container ID** (ID ở cột đầu tiên
  output của `crictl ps`).
- Sau khi stop: `kubectl get pods nginx2` → **RESTARTS `1 (15s ago)`** —
  container vừa được restart vì `restartPolicy: Always` và "something bad
  happened to the container".

*(Transcript ghi "Popman" và dùng "grab" cho việc lọc output — chuẩn hóa thành
Podman và `grep`.)*

## Phần 5 — Jobs

### Khái niệm

- Job chạy pod với **`restartPolicy: Never`** — ý tưởng: **chạy đúng một lần**
  rồi kết thúc (runs to completion).
- Muốn pod chạy đến hoàn thành → dùng **job**, không dùng pod thường.
- Job hữu ích cho **one-shot tasks**: backup, calculation, batch processing...
- Nên set **`spec.ttlSecondsAfterFinished`** (vd 60): thiếu nó job **không tự
  dọn dẹp** sau khi xong — job là sự kiện một lần nên thường muốn nó tự biến
  mất sau một khoảng thời gian.

### Các loại job — completions & parallelism

| Loại job | `completions` | `parallelism` | Hành vi |
|---|---|---|---|
| Non-parallel | 1 | 1 | Một pod chạy; pod fail thì chạy lại |
| Parallel — fixed completion count | N | M (vd 3 / 10) | Job hoàn thành khi đủ N lần chạy thành công |
| Parallel — work queue | 1 | tùy ý | Nhiều pod chạy; **một** pod thành công là job hoàn thành |

### Demo — job cơ bản

> **Mẹo thi:** `-h` (help) của lệnh `kubectl create ...` có **ready-to-use
> examples nằm ngay đầu output** — đừng quên.

```bash
kubectl create job onejob --image=busybox -- date
kubectl get jobs,pods
```

- Job **tạo pod** — đó là việc job làm; pod mang **suffix tự sinh** trong tên
  → nhận biết pod sinh ra từ job nào.
- `onejob` đã **Complete** ngay sau khi chạy.
- `kubectl get pods <onejob-pod> -o yaml | grep restartP` → `restartPolicy:
  Never` — đúng định nghĩa job.
- `kubectl delete job onejob` — không delete thì không ai delete giúp.

### Demo — job nâng cao qua YAML

```bash
kubectl create job mynewjob --image=busybox --dry-run=client -o yaml \
  -- sleep 5 > mynewjob.yaml
```

- Lý do ghi ra file: cần thêm parameters **khó đưa qua command line** — làm
  thứ nâng cao trong Kubernetes thì **generate YAML rồi thêm properties vào
  file**; trên kỳ thi việc này "shouldn't give you any problem at all".
- Sửa `mynewjob.yaml` — phân biệt 2 mức specification:
  - `spec` **top-level** = **job specification** — nơi thêm job properties.
  - `spec` dưới `template` = **pod specification** — dành cho pod properties,
    không dùng ở đây.
  - Thêm vào **job spec**: `completions: 3` và `ttlSecondsAfterFinished: 60`.

```bash
kubectl apply -f mynewjob.yaml
kubectl get jobs,pods    # completions 0/3 → 2/3 → 3/3 (từng pod hoàn thành)
```

- Chờ đúng 60 giây (TTL) → `kubectl get jobs,pods` → job **và** các pod của nó
  **đã tự bị xóa**.
- Khuyến nghị của tác giả: dùng job thì **luôn** set `ttlSecondsAfterFinished`.

*(Transcript ghi "if you are going to use a pod, you should also use
ttlSecondsAfterFinished" — theo ngữ cảnh đúng là "use jobs"; "pipe grab
restart p" chuẩn hóa thành `| grep restartP`.)*

## Phần 6 — Cron Jobs

### Khái niệm

- Job chạy task **một số lần cụ thể**; **cron job = job + schedule**.
- Schedule dùng **crontab syntax của Linux** — từng dùng crontab trên Linux thì
  scheduling cron job trên Kubernetes sẽ dễ.
- Chuỗi hoạt động: **cron job → job → pod**. Chạy cron job thì một job được
  schedule; job đó, lượt nó, start pod.

### Test cron job không cần chờ lịch

```bash
kubectl create job <tên-job> --from=cronjob/<tên-cronjob>
```

- Trigger cron job chạy **ngay tại chỗ** — tiện khi cần verify trên kỳ thi mà
  cron job chỉ chạy **một lần mỗi ngày**.

### Schedule — 5 vị trí crontab

```text
* * * * *
| | | | |
| | | | └─ day of week
| | | └─── month
| | └───── day of month
| └─────── hour
└───────── minute
```

- `*/1` là **redundant** — `*` thôi đã nghĩa là "any" (mỗi phút).
- `*/2` = mỗi 2 phút.
- **Đếm kỹ cho đủ 5 vị trí** trước khi Enter.

### Demo

```bash
kubectl create cronjob -h | less      # examples nằm ở đầu output
kubectl create cronjob runme --image=busybox --schedule="*/2 * * * *" \
  -- echo greetings from the cluster
kubectl get cronjobs,jobs,pods
```

- Trong `-h`: example đầu **không hợp lý** (busybox không kèm command); example
  thứ hai có `-- date` ở cuối mới dùng được.
- `kubectl get cronjobs,jobs,pods` thấy đủ chuỗi: cronjob `runme` (LAST
  SCHEDULE **2s** — demo may mắn bắt đúng nhịp) → job `runme-<suffix>` đang
  running → pod **ContainerCreating**.
- Trigger thêm: `kubectl create job runme --from=cronjob/runme`.
- Verify kết quả bằng `kubectl logs <runme-pod>` → `greetings from the cluster`
  — ứng dụng trong Kubernetes **không nối với standard output nào**, phải dùng
  `kubectl logs` mới biết chúng đã làm gì.
- Để yên thì cron job **cứ sinh thêm job** theo lịch → dọn bằng cách xóa
  top-level: `kubectl delete cronjob runme` xóa **cả chuỗi** — cron job, các
  job đã tạo, và các pod do job schedule. Get lại: sạch.

*(Transcript ghi "run me", "minus minus from is cron job" và "start slash one"
— chuẩn hóa thành `runme`, `--from=cronjob/` và `*/1`; bản ghi "hour for
minute" ở vị trí đầu là lỗi transcribe — trường đầu tiên của schedule là
**minute**.)*

## Phần 7 — Clean up resources

### Nguyên tắc dọn dẹp

- Dọn dẹp resource **không tự động xảy ra**. Một số resource có option tự dọn
  — vd chạy pod với **`--rm`** để tự remove khi xong; còn lại phải **định kỳ
  dọn tay**.
- **Quy tắc generic:** resource này tạo ra resource khác (cron job → job →
  pod; deployment → pod) → **xóa resource top-level**, mọi thứ bên dưới được
  xóa theo.
- Xóa resource **lower-level** (vd pod do deployment quản lý) → nó **được tạo
  lại ngay**. Pod do deployment quản lý thì phải xóa **deployment**, không
  xóa pod.

### Force delete — tuyệt đối tránh

- Có option force delete — tác giả **khuyên mạnh mẽ đừng dùng**: resource có
  thể kẹt mãi ở state **deleting** → không tạo lại được resource đó nữa.
- Trên kỳ thi: force delete → resource stuck → **fail mất phần đề đó** —
  *"you might fail your exam because of it"*.
- Lệnh "terrible" (biết mặt để tránh, không chạy):

```bash
kubectl delete <resource> --force --grace-period=-1
```

- Warning khi chạy: immediate deletion **không đợi confirmation** rằng
  resource đã terminate — resource **có thể tiếp tục chạy vô thời hạn** trong
  cluster. Không phải điều ai muốn.

### Demo

```bash
kubectl delete all            # KHÔNG xóa gì — không có tên resource
kubectl delete all --all      # xóa pods, services, deployments (namespace hiện tại)
kubectl get all               # service "kubernetes" đã tự xuất hiện lại
```

- `kubectl delete all` thiếu `--all` → không resource nào được chỉ định →
  không xóa gì.
- `--all` xóa sạch namespace hiện tại, kể cả service `kubernetes` — essential
  service của Kubernetes, nhưng **tự động được tạo lại**.
- **Tuyệt đối không thêm `-A`:** xóa trên **mọi namespace** — kéo theo cả
  **system resources**. Trong demo tác giả gõ tới đó rồi nhấn **Ctrl+U** để
  xóa sạch dòng lệnh và không chạy.

*(Transcript ghi "Pult", "//rm", "minus a", "control U" và "you will fill that
part of the exam" — chuẩn hóa thành pod, `--rm`, `-A`, Ctrl+U và "fail".)*

## Phần 8 — End-of-lesson lab

### Đề bài

Tạo **cron job chạy lệnh `sleep 30`**. Nếu sau **15 giây** mà job vẫn đang
chạy, nó phải **bị kill** — và chu kỳ bắt đầu lại. Đề nghe "không có nghĩa"
nhưng có: mục đích là **test các parameter** và verify chúng hoạt động thật.

### Phân tích trước khi làm

Yêu cầu "vẫn chạy sau 15 giây thì bị kill" → cần property **deadline/
terminate cho job**. Tìm bằng `kubectl explain` — đi từng mức spec:

```bash
kubectl explain cronjob.spec                    # chỉ có timing properties
kubectl explain cronjob.spec.jobTemplate.spec   # thấy activeDeadlineSeconds
```

- Trong `cronjob.spec` chỉ thấy properties về **timing** — vd
  `startingDeadlineSeconds` (*optional deadline in seconds for starting the
  job if it misses scheduled time for any reason*) — không phải cái cần, và
  không có property nào kill job.
- Lý do: **cron job properties là về lịch chạy**; property điều khiển job
  nằm ở **job template**.
- `cronjob.spec.jobTemplate.spec` có **`activeDeadlineSeconds`** — *duration
  in seconds relative to the start time that the job may be continuously
  active before the system tries to terminate it* → chính xác yêu cầu đề
  bài.

### Lời giải

Generate YAML trước — property này **chỉ thêm được qua file**:

```bash
kubectl create cronjob sleepy --image=busybox --schedule="* * * * *" \
  --dry-run=client -o yaml -- sleep 30 > lab6.yaml
```

- Đề không chỉ định schedule → chọn **mỗi phút** (`* * * * *` — đủ 5 vị trí).
- Đừng quên `-- sleep 30` — lệnh pod sẽ chạy.

Sửa `lab6.yaml` — phân biệt **3 mức spec** trong file:

```text
spec:                    # cronjob specification (schedule...)
  jobTemplate:
    spec:                # job specification ← THÊM activeDeadlineSeconds: 15
      template:
        spec:            # pod specification (containers...)
```

- Thêm **`activeDeadlineSeconds: 15`** vào `spec.jobTemplate.spec` — đúng
  mức vừa tìm thấy bằng explain.

Chạy và verify:

```bash
kubectl apply -f lab6.yaml
kubectl get all           # cronjob/batch sleepy xuất hiện
kubectl get all           # chờ đầu phút: job chạy → pod chạy → Terminating
```

- Cron job mỗi phút thường chạy ở **đầu phút** (top of the minute) → chờ vài
  giây.
- Demo: cronjob start job, pod chạy được 6 giây ở lần get đầu; lần get sau pod
  đã **Terminating** (pod ~19 giây tuổi) — `sleep 30` chưa xong mà job đã bị
  system terminate. Đó chính là hiệu ứng `activeDeadlineSeconds: 15`; phút kế
  tiếp chu kỳ lặp lại. Mission accomplished.
- Nhận xét của tác giả: chạy sleep 30 rồi kill ở 15 giây thì **lạ**, nhưng đó
  là cách hay để **test parameter và verify chúng thật sự hoạt động**.

*(Transcript ghi "cronjob.spec pipelines", "minus O YAML", "minus minus sleep
30" và "jobtemplate.spec" — chuẩn hóa thành `| less`, `-o yaml`, `-- sleep 30`
và `cronjob.spec.jobTemplate.spec`.)*
