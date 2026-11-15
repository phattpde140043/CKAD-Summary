# Bài 17 — Observability

Bài học cuối cùng của course trước Module 7 (sample exam) —
**observability**: khái niệm logs/metrics/traces, API health endpoints,
và cách dùng **probes** để Kubernetes tự monitor ứng dụng — kết thúc bằng
lab thực hành với probes.

## Lộ trình bài học

1. Understanding observability
2. API deprecations and using Kubernetes API health endpoints
3. Using probes to monitor applications
4. End-of-lesson lab

## Phần 1 — Understanding observability

- **Observability** = monitoring trạng thái của hệ thống bằng cách
  kiểm tra **external outputs** của nó — **logs, metrics, traces**.
- Là thứ thực sự cần trong cloud environment vốn phức tạp; có thể coi
  là "monitoring on steroids", tập trung vào việc xử lý **tình huống
  bất ngờ** (unexpected situations).
- Các công cụ observability phổ biến (cả nhóm do **ecosystem** cung
  cấp, **không thuộc vanilla Kubernetes**):

| Công cụ | Tập trung vào |
|---|---|
| Prometheus, Grafana | Metrics |
| ELK stack, Loki | Logs |
| Jaeger, Zipkin | Tracing |

**Không nằm trong đề CKAD** — các công cụ trên chỉ cần biết tên là đủ,
đề không hỏi về chúng.

## Phần 2 — API deprecations and using Kubernetes API health endpoints

- External applications không có sẵn trong vanilla Kubernetes, nhưng
  Kubernetes có **probes** — và probes làm việc với **API health
  endpoints**. *(Về API deprecations — nhắc lại Bài 14 Phần 4, đã xử lý
  kỹ ở đó.)*
- API health endpoints được lập trình vào một số ứng dụng để monitor
  xem ứng dụng còn hoạt động như mong đợi hay không. Common practice:
  ứng dụng cung cấp access tới **`/healthz`** để test application
  availability.
- Chính **kube-apiserver** exposure 3 endpoints để test bản thân nó:
  - `/healthz` — trả `OK` nếu API server healthy.
  - `/livez` — cho biết API server còn alive.
  - `/readyz` — cho biết API server đã ready để phục vụ requests.
- Test từ command line: `curl -k https://$minikubeip:8443/healthz` —
  kết quả phải là `OK`.
- Bất kỳ ứng dụng web-based nào cũng có thể cung cấp endpoints tương
  tự — không phải universal standard, nhưng là standard gặp rất nhiều:
  khả năng cao dùng được với ứng dụng khác.

### Demo — curl API health endpoint

```bash
curl -k https://$minikubeip:8443/healthz
# OK
curl -k https://$minikubeip:8443/bogus
# forbidden: cannot get path "bogus"
```

- `$minikubeip` là biến shell được thay bằng địa chỉ Minikube IP thật.
- Phải nhìn kỹ: `OK` nằm ngay trước prompt — đó là cách Kubernetes
  endpoint trả lời rằng nó đang hoạt động.
- So sánh: curl tới path không tồn tại (`bogus`) → `forbidden, cannot
  get path` — không hoạt động.
- Các endpoint `healthz` này (và các endpoint khác) chính là thứ
  **Kubernetes probes** sử dụng — nội dung của Phần 3.

## Phần 3 — Using probes to monitor applications

- Probes cung cấp **common solution** để monitor applications.
- Probe là **một test đơn giản được định nghĩa như container property**
  — trong nhiều trường hợp là một command.
- Probe dùng để test xem ứng dụng sử dụng nó còn functional không; probe
  không phản hồi → ứng dụng được **restart** — coi như một failure.

**Ba probe test types** (cấu hình trong `pod.spec.containers`):

| Kiểu test | Cách hoạt động |
|---|---|
| `exec` | Chạy một command — kỳ vọng trả về exit value 0 |
| `httpGet` | HTTP request phải trả response code 200–399 — indicator HTTP thành công |
| `tcpSocket` | Test connectivity tới TCP socket (còn gọi là port) |

- Probes cấu hình được với **failureThreshold** để quyết định ứng dụng
  được bao lâu để phản ứng — đôi khi nên cho ứng dụng một chút thời
  gian.

**Ba probe types** — test ứng dụng ở các phase khác nhau của lifetime:

| Probe | Mục đích | Khi fail |
|---|---|---|
| `livenessProbe` | Check ứng dụng còn alive | Container bị **restart** |
| `readinessProbe` | Check ứng dụng ready phục vụ requests | Container bị **remove khỏi danh sách services available** (không restart) |
| `startupProbe` | Verify initial startup — hữu ích khi startup chậm | Các probe khác **không chạy** trước khi probe này thành công |

> **Ghi nhớ thi CKAD**: chỉ **liveness** mới restart container;
> **readiness** fail chỉ rút pod khỏi service endpoints; **startup**
> chặn các probe khác cho tới khi start xong.

### Demo — readiness probe với file không tồn tại

File `busybox-ready.yaml` (course git repo) tạo pod `busybox-ready` —
standard pod, điểm quan tâm là readinessProbe: period 10 giây, dùng
command `cat /tmp/nothing` — check file đó có tồn tại không (dĩ nhiên
là chưa có → probe phải fail):

```bash
kubectl apply -f busybox-ready.yaml
kubectl get pods                          # busybox-ready 0/1, Running
kubectl describe pod busybox-ready
# Readiness probe failed: can't open /tmp/nothing: No such file or directory
kubectl exec -it busybox-ready -- touch /tmp/nothing
kubectl get pods                          # 1/1
```

- Pod **Running nhưng 0/1**: chạy rồi nhưng "không hoàn toàn hạnh phúc"
  — đợi hết period 10 giây cũng vẫn 0/1.
- Điểm gây nhầm: **Running** có thể gợi ý rằng mọi thứ ổn — chưa chắc;
  phải nhìn cột **READY** và tìm hiểu chuyện gì đang xảy ra.
- `kubectl describe` cho thấy ở dòng cuối output: readiness probe
  failed — không mở được `/tmp/nothing`, không có file này.
- Pod vẫn đang chạy nên có thể can thiệp: `kubectl exec` + `touch` tạo
  luôn file đó. Test là verify file `/tmp/nothing` tồn tại — giờ nó tồn
  tại → `kubectl get pods` thấy **1/1**.

## Phần 4 — End-of-lesson lab — Probes

### Đề bài

Chạy pod tên **probed**, start image **nginx:latest**, dùng probes đáp
ứng các điều kiện:

1. Ứng dụng có **30 giây để start**.
2. Sau khi start, **mỗi 10 giây** test rằng ứng dụng vẫn healthy.
3. Nếu ứng dụng không còn healthy → **restart**.
4. Probe thực hiện bằng cách **test access tới Kubernetes healthz
   endpoint**.

Từ lý thuyết Phần 3: điều kiện 1 → `startupProbe`; điều kiện 2 + 3
(restart khi fail) → `livenessProbe` với period 10 giây — chỉ liveness
mới restart, readiness chỉ remove khỏi service endpoints.

### Lời giải

- Lab này có thể là một trong những thứ khó hơn nếu gặp trên đề thi —
  và là dịp luyện thói quen **quay về documentation**: tìm "probes" →
  trang **Configure Liveness, Readiness and Startup Probes**, nơi có
  sẵn các YAML example:
  - Liveness probe với **exec** command — cung cấp vài parameters cần
    thiết, điểm khởi đầu tốt nhưng chưa đủ.
  - **Liveness HTTP request** — tiến gần hơn nữa: HTTP request tới
    healthz endpoint.
  - TCP liveness probe và gRPC — không phải cái cần ở đây.
- Dựa trên các example đó để ra kết quả. File mẫu trong course repo:
  **healthz-probe.yaml** — pod `readiness-exec` (tên riêng của file mẫu;
  trên đề thi đặt tên đúng theo đề bài), probe **exec** chạy
  `wget --no-check-certificate --spider` trỏ tới Kubernetes API
  endpoint.

```bash
kubectl apply -f healthz-probe.yaml
kubectl get pods                          # readiness-exec 0/1 → chờ → 1/1
```

- Vì sao `exec` chứ không phải probe type khác? Vì exec **đơn giản**:
  chạy được Linux command nào là viết được probe đó; các probe types
  khác khó hơn một chút. Cuối cùng, thứ counts là **nó hoạt động**.
- Có timeouts liên quan nên lúc đầu 0/1 — cho nó chút thời gian, sau đó
  **1/1 running**: đây chính là thứ lab yêu cầu.

## Tổng hợp lệnh Bài 17

| Lệnh | Chức năng |
|---|---|
| `curl -k https://$minikubeip:8443/healthz` | Test API health endpoint của kube-apiserver (tương tự `/livez`, `/readyz`) |
| `kubectl get pods` | Chú ý cột READY — 0/1 nghĩa là probe chưa pass dù pod Running |
| `kubectl describe pod <pod>` | Events cuối output cho thấy probe failed kèm lý do |
| `kubectl exec -it <pod> -- touch /tmp/nothing` | Can thiệp trong pod để làm probe pass trong demo |
| `kubectl apply -f healthz-probe.yaml` | Triển khai pod kèm probes từ file YAML |

> Đó là bài học cuối cùng của phần lý thuyết. Module 7 — **Sample
> exam** — tự kiểm tra: làm bài mô phỏng đề thi thật rồi dùng script
> chấm điểm của course để biết mình đã sẵn sàng chưa trước khi thi CKAD.
