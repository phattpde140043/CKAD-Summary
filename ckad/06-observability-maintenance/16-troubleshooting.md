# Bài 16 — Troubleshooting

Bài đầu tiên của Module 6 — **troubleshooting Kubernetes**: phương pháp
tổng quát để chẩn đoán sự cố, phân tích ứng dụng đang fail, xử lý sự cố
network access, khai thác cluster event logs, troubleshoot authentication
issues, debug ứng dụng bằng Linux skills — kết thúc bằng end-of-lesson lab.

## Lộ trình bài học

1. Understanding generic troubleshooting
2. Analyzing failing applications
3. Troubleshooting network access
4. Using cluster event logs
5. Troubleshooting authentication issues
6. Debugging Kubernetes applications
7. End-of-lesson lab

## Phần 1 — Understanding generic troubleshooting

Troubleshooting bắt đầu từ việc hiểu **chuyện gì đang diễn ra** — từ YAML
tới lúc user truy cập ứng dụng là một chuỗi mắt xích, và mỗi mắt đều có
thể hỏng.

**Luồng triển khai:**

1. YAML được đưa tới kube-apiserver qua `kubectl`.
2. API server **syntax check trước khi ghi vào etcd** — lỗi syntax báo
   về ngay lập tức.
3. Resource ghi vào etcd kèm vài checks — nơi xuất hiện trạng thái
   **Pending** hoặc generic error khác.
4. **Scheduler** pick up và tìm **eligible node** cho pod.
5. Image được download về node — lỗi ở bước này thường do thiếu
   credentials lấy image từ registry.
6. Container start trong pod → có ứng dụng chạy.

**Luồng truy cập của user:** user → DNS → ingress → service → pod.
Service dùng **labels** xác định pod nào nó phục vụ; ingress nối request
tới service bằng service name *(nhắc lại Module 4)*.

Cách phân tích sự cố tốt nhất: **đi theo user request**:

- User → DNS: **name lookup error** → lỗi DNS, nằm **ngoài**
  Kubernetes.
- DNS → ingress: virtual host definition phải route được request về
  đúng pod (qua chuỗi service + labels) — không tới được thì trước hết
  troubleshoot **labels** của application.

Chiều ngược lại (phía triển khai): lỗi trong YAML khiến scheduler không
tìm được eligible node; **image download error** nghĩa là có thể thiếu
credentials fetch image từ image registry *(nhắc lại Bài 12)*.

## Phần 2 — Analyzing failing applications

- Sau `kubectl create`/`kubectl run`: resource ghi vào **etcd** →
  **kube-scheduler** tìm node → image pull về node → container start và
  chạy **entry point application**. Thành công hay thất bại của entry
  point, cùng **restart policy**, quyết định hành động tiếp theo.

Pod trong vòng đời đi qua các trạng thái:

| Trạng thái | Ý nghĩa |
|---|---|
| **Pending** | Pod đã được API server validate, có entry trong etcd, nhưng điều kiện tiên quyết chưa đạt — điển hình: scheduler chưa tìm được eligible node mà YAML vẫn OK |
| **Running** | Pod đang chạy thành công |
| **Completed** | Pod đã hoàn thành công việc của nó |
| **Failed** | Pod đã kết thúc nhưng có gì đó sai |
| **CrashLoopBackOff** | Pod đã fail và theo restart policy được cluster khởi động lại — liên tục |
| **Unknown** | Không lấy được pod status |

- `kubectl get pods` — **việc đầu tiên** làm khi bất cứ thứ gì fail.
- Vào trạng thái fail → `kubectl describe` để xem application state và
  góc nhìn của cluster về nó:
  - Đọc **events** ở cuối output trước, rồi mới tới application state.
  - Check **Last State** và đặc biệt **exit code** của application:
    - **Exit code 0** — ứng dụng hoàn thành thành công, không cần điều
      tra thêm: từ góc nhìn application không có vấn đề.
    - **Exit code ≠ 0** — vấn đề nằm ở entry point application →
      `kubectl logs`.
  - Pod restart liên tục → `kubectl logs --previous` — xem logs của
    container instance trước đó (không còn chạy).

### Demo — busybox thoát với exit code 0

```bash
kubectl create deploy failureone --image=busybox
kubectl get pods                          # CrashLoopBackOff
kubectl describe pod failureone-<suffix>
```

- State **terminated**, reason **completed**; Last State tương tự —
  ứng dụng đang làm đúng việc nó phải làm.
- Đây là hiện tượng điển hình của image Linux system vốn mặc định start
  **bash shell**: bash trong môi trường Kubernetes không nối với
  standard output nên thoát ngay với **exit code 0**.
- Từ góc nhìn application không có gì sai; từ góc nhìn user thì có. Khắc
  phục: start ứng dụng kèm một **command**, hoặc đặt `restartPolicy:
  Never`. Vì không có gì hỏng nên cũng chẳng có gì để sửa — dọn demo:

```bash
kubectl delete deploy failureone
```

### Demo — mariadb exit code 1

```bash
kubectl create deploy failuretwo --image=mariadb
kubectl get pods                          # ContainerCreating → CrashLoopBackOff
kubectl describe pod failuretwo-<suffix>  # terminated, reason error, exit code 1
kubectl logs failuretwo-<suffix>          # hiện lỗi của ứng dụng
```

- Lúc đầu pod kẹt `ContainerCreating` — image mariadb chưa download
  xong; download xong mới chuyển `CrashLoopBackOff`.
- **Exit code 1** → vấn đề nằm ở entry point application, không phải
  Kubernetes → `kubectl logs` cho thấy lỗi cụ thể.

*(nhắc lại Bài 12)* Với image mariadb, lỗi điển hình là ứng dụng đòi
biến môi trường `MARIADB_ROOT_PASSWORD` — cấp env là pod chạy.

- Nguyên tắc: **troubleshooting ứng dụng fail luôn bắt đầu từ pod,
  không bắt đầu từ deployment**.

## Phần 3 — Troubleshooting network access

- Chuỗi truy cập: user qua **DNS** → **Ingress** (rules nối incoming
  request tới service bằng **service name**) → **service** nối tới pods
  dựa trên **labels và selectors**.
- Troubleshoot access theo đúng thứ tự đó:
  1. DNS request có tới Ingress không?
  2. Có Ingress rule nối incoming request tới service không?
  3. Service có nối tới pods với đúng labels không?
- Muốn test process đã chạy như mong đợi trong pod nhưng không truy cập
  được từ ngoài: **port forwarding** — `kubectl port-forward myweb
  8080:80` để truy cập pod ở cổng 8080.

### Demo — curl myapp.local không kết nối được

```bash
curl myapp.local                          # couldn't connect to server
ping myapp.local                          # OK — name vẫn resolve
kubectl get ing                           # có ingress, host myapp.local
kubectl describe ing myingress           # xem backends — lấy từ service
kubectl get svc                           # somedeploy — ClusterIP
curl 10.96.20.202                         # welcome to nginx
```

- `ping` được → phía tên/virtual host ổn. Backends trong ingress lấy từ
  service; service là **ClusterIP** nên curl tới IP đó từ node trong
  cluster — **thành công** → service reachable, backend IP đúng.
- Còn đúng một nghi phạm: **Ingress Controller**.

```bash
kubectl get ns                            # không thấy namespace ingress controller
kubectl get all -n kube-system            # cũng không thấy
minikube addons list                      # ingress disabled
minikube addons enable ingress
curl myapp.local                          # hoạt động
```

- Trên Minikube ingress controller quản lý qua **add-on** — đang
  disabled nên cả chuỗi chết ở đó. Enable, chờ khởi động xong là curl
  được.
- Kết luận: chuỗi **DNS → ingress controller → ingress resource →
  service → backend pods** — một mắt xích bị hiccups là không work.

### NetworkPolicy

- **NetworkPolicy** là firewall *(nhắc lại Module 4)*: policy áp lên pod
  thì chỉ traffic được phép mới đi qua; policy áp mà không cho phép gì
  thì không work.
- Check policy có áp lên pod không: verify `spec.podSelector` của
  NetworkPolicy; nếu có, phía `spec.ingress` cần `podSelector.
  matchLabels` chỉ định pods nào được truy cập.

### Demo — NetworkPolicy chặn wget

```bash
kubectl exec -it busybox -- wget --spider --timeout=1 nginx
# connecting to nginx (ClusterIP) ... download timed out
kubectl get networkpolicy                 # access-nginx
kubectl describe networkpolicy access-nginx
kubectl get pods --show-labels
kubectl label pod busybox access=true
kubectl exec -it busybox -- wget --spider --timeout=1 nginx
# remote file exists
```

- "connecting to nginx" kèm cluster IP → **name resolution hoạt động**;
  nhưng "download timed out" → khả năng cao chuyện do NetworkPolicy.
- `describe networkpolicy access-nginx`: podSelector `app=nginx`; allow
  ingress traffic từ pods có label `access=true`.
- `--show-labels`: pod nginx có `app=nginx`; busybox chỉ có `app=sleepy`
  — thiếu `access=true` nên bị chặn.
- Gán label xong wget ngay lập tức thành công ("remote file exists") —
  đúng là NetworkPolicy.

### Network add-on

- Kubernetes chạy với một **network add-on**; đổi add-on có thể break
  hoặc fix network problems. **Flannel không hỗ trợ NetworkPolicy** —
  khuyến nghị dùng **Calico** cho đầy đủ tính năng.
- Check add-on đang dùng:

```bash
kubectl get all -n kube-system
# calico-node (DaemonSet) + calico-kube-controllers (Deployment + ReplicaSet)
```

- Thấy calico-node và calico-kube-controllers là đang dùng add-on tốt.
  Không có thì phải cài lại: Minikube không có add-on toggle cho CNI —
  bring cluster down rồi up với tùy chọn CNI:

```bash
minikube delete
minikube start --cni=calico
```

## Phần 4 — Using cluster event logs

- **Cluster event log** — nguồn thông tin hữu ích khi có trouble.
- `kubectl get events` — overview các event toàn cluster;
  `kubectl get events -o wide` — chi tiết hơn nữa.
- Chưa rõ sự cố liên quan tới resource nào → `kubectl get events` (cần
  overview); đã biết resource → `kubectl describe` tốt hơn.

### Demo — kubectl get events

```bash
kubectl get events
kubectl get events -o wide
kubectl get events | wc -l                # 59 events
```

- Output khá overwhelming — 59 events trong demo — nhưng khi đã mò
  không ra nguyên nhân thì đây là chỗ có thể ra manh mối.

## Phần 5 — Troubleshooting authentication issues

- Triệu chứng: **không get access vào cluster**.
- Access vào cluster mặc định cấu hình qua file **`.kube/config`** —
  file này được copy từ control node, nơi nó lưu tại
  **`/etc/kubernetes/admin.conf`**.
- `kubectl config view` — xem nội dung của local authentication file
  *(nhắc lại Bài 14 Phần 1)*.
- Authorization-related problems → `kubectl auth can-i` *(nhắc lại Bài
  15 Phần 1)* — ví dụ `kubectl auth can-i create pods`.
- Authentication issues khó reproduce; nếu gặp thì kiểm tra
  `.kube/config` trước hết — đa số vấn đề nằm đúng ở đó.

## Phần 6 — Debugging Kubernetes applications

- Khi `kubectl describe` và `kubectl logs` **không đưa ra thêm insight**
  về application failure → **`kubectl exec`** chạy commands ngay trong
  pod.
- `kubectl exec -it <pod> -- /bin/bash` mở shell trong pod; bash không
  có thì dùng `/bin/sh`. Dùng bất kỳ Linux command nào để phân tích
  container đang cố gắng làm gì.
- Ý thức trước: một số commands muốn dùng có thể **không có** trong
  container.
- Pod phải đang **running** mới exec phân tích được — làm vậy trên pod
  `CrashLoopBackOff` là vô nghĩa.

### Demo — debug nginx pod qua /proc

```bash
kubectl exec -it nginx -- sh
ps                                       # ps: not found
ls                                       # có
ls /proc                                 # 1 7 8 16 ... — mỗi số là một PID
cd /proc/1
cat cmdline                              # nginx master process nginx -g daemon off
```

- `sh` là **Bourne shell** — shell tương thích nhất, thay cho bash.
- `ps` không có, `ls` có: container phục vụ **một mục đích duy nhất** —
  cung cấp môi trường chạy ứng dụng đúng cách nó cần được start — nên
  không cần utilities như ps.
- Nhưng cũng không phải bỏ cuộc: có Linux skills thì vẫn điều tra được
  qua **`/proc`** — thông tin mọi process đang chạy. Số cao nhất luôn là
  command vừa khởi động gần nhất (lần sau 17, 18...).
- `/proc/<pid>/cmdline` cho thấy chính xác process chạy gì — ở đây là
  nginx master process (`nginx -g daemon off`). Còn nhiều file khác
  trong các thư mục này — nơi để đi thật sâu, nhưng không có Linux
  expertise thì vào đây chẳng để làm gì.
- Tin tốt: đa số trường hợp, **standard Kubernetes tools là đủ**.

## Phần 7 — End-of-lesson lab — Troubleshooting

### Đề bài

Chạy script `lab16break.sh` (có trong Git repo của course), sau đó dùng
`kubectl get all` và sửa vấn đề tìm thấy.

### Lời giải

```bash
./lab16break.sh
kubectl get all
# couldn't get current server API group list ... localhost:8080
```

- Lỗi nhắc **localhost:8080** — "ngửi" liền ra authentication problem:
  client không biết phải đi đâu nên thử localhost.
- Authentication được lo qua `~/.kube/config` — mở file ra thấy dòng
  **`current-context`** bị **comment (#)**.
- Context là nơi định nghĩa **user + cluster + current namespace**;
  không có current context thì Kubernetes client không biết nơi nào để
  kết nối.
- Sửa: bỏ dấu `#` trước dòng `current-context:` → `kubectl get all`
  chạy lại bình thường. Chẳng có gì ngoài phạm vi đã biết.

## Tổng hợp lệnh Bài 16

| Lệnh | Chức năng |
|---|---|
| `kubectl get pods` | Trạng thái pod — việc đầu tiên khi có sự cố |
| `kubectl describe pod` | Events (cuối output), Last State và exit code của application |
| `kubectl logs` / `kubectl logs --previous` | Logs ứng dụng; `--previous` cho container instance trước đó |
| `kubectl port-forward <pod> 8080:80` | Test process trong pod mà không cần qua service/ingress |
| `kubectl exec -it <pod> -- sh` | Mở shell trong pod để debug — bash không có thì dùng sh |
| `kubectl get events` (`-o wide`) | Overview event toàn cluster khi chưa biết resource nào liên quan |
| `kubectl config view` | Xem `.kube/config` — nơi đa số authentication issue nằm |

> Troubleshooting là phản ứng khi sự cố đã xảy ra. Bài 17 —
> **Observability** — phần còn lại của Module 6, chuyển sang phía chủ
> động: quan sát ứng dụng để phát hiện vấn đề trước khi nó nổ ra.
