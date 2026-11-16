# Deep dive — Job & CronJob nâng cao

> Cấp độ mục tiêu: **Middle**. Toàn bộ nội dung *(bổ sung từ nghiên cứu
> ngoài video)*; tạo Job/CronJob cơ bản đã có *(nhắc lại Bài 6)*.

## ① Concept

Job sinh ra để **chạy tới khi thành công** — bộ field dưới đây quyết định
hành vi khi fail và khi chạy quá lâu:

- **backoffLimit** (mặc định 6) — số lần thử lại tối đa khi pod fail;
  hết lượt → Job **Failed** với reason `BackoffLimitExceeded`. Mỗi lần
  thử lại chờ tăng dần: 10s → 20s → 40s (tối đa 6 phút).
- **activeDeadlineSeconds** — giới hạn **tổng thời gian sống** của Job
  (tính cả các lần retry); hết hạn → pod bị kill, Job **Failed** với
  reason `DeadlineExceeded` *(đã dùng trong Lab 6.8)*.
- **completions / parallelism** — completions: tổng số pod phải thành
  công; parallelism: chạy đồng thời bao nhiêu. Mặc định 1/1 — Job đơn.
- **ttlSecondsAfterFinished** — Job tự xóa sau N giây Complete/Failed
  *(đã dùng trong Lab 6.5)*.
- **suspend** (Job, GA v1.24) — `true` = đóng băng: pod đang chạy bị
  xóa; mở lại (`false`) → pod được tạo lại để chạy tiếp phần chưa xong.

CronJob bọc Job trong `jobTemplate` nên dùng được đủ field trên, cộng
thêm các field lịch chạy:

- **timeZone** (GA v1.27) — schedule hiểu theo timezone đặt tên (vd
  `Asia/Ho_Chi_Minh`); không set thì theo giờ của
  kube-controller-manager (Minikube: UTC).
- **concurrencyPolicy** — hành xử khi tới lịch mà Job trước chưa xong:
  `Allow` (mặc định — chạy chồng), `Forbid` (bỏ lần mới), `Replace`
  (hủy Job đang chạy, chạy Job mới).
- **startingDeadlineSeconds** — Job phải start trong khoảng này kể từ
  lúc đến lịch; lỡ quá → bỏ luôn lần chạy đó.
- **successfulJobsHistoryLimit / failedJobsHistoryLimit** (mặc định
  3/1) — giữ lại bao nhiêu Job cũ để `get jobs` và `logs` tra cứu.

`restartPolicy` trong pod template: `Never` → mỗi lần thử là **pod
mới** với tên khác (dễ quan sát, backoffLimit đếm pod fail);
`OnFailure` → restart container tại chỗ trong cùng pod.

## ② Configuration (YAML)

CronJob "đủ giáp" cho batch production:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "30 2 * * *"          # 02:30 hằng ngày
  timeZone: Asia/Ho_Chi_Minh      # GA v1.27
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 300
      ttlSecondsAfterFinished: 120
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: report
            image: busybox
            command: ["sh", "-c", "echo report && sleep 60"]
```

Job batch song song:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-process
spec:
  completions: 6                  # cần 6 pod thành công
  parallelism: 2                  # chạy 2 pod cùng lúc
  backoffLimit: 4
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo item done"]
```

## ③ Command (kubectl)

```bash
# Job fail lập tức để quan sát retry
kubectl create job failjob --image=busybox -- false
kubectl get jobs,pods -w
kubectl describe job failjob
# Events: BackoffLimitExceeded — "Job has reached the specified
# backoff limit"

# liệt kê pod của một Job theo label tự sinh
kubectl get pods -l job-name=failjob

# đóng băng CronJob (maintenance window)
kubectl patch cronjob ticker -p '{"spec":{"suspend":true}}'
kubectl get cronjobs              # cột SUSPEND = true
```

Field nâng cao không có sẵn trong imperative → sinh YAML rồi sửa *(nhắc
lại cách làm Lab 6.8)*:

```bash
kubectl create cronjob ticker --image=busybox \
  --schedule="* * * * *" --dry-run=client -o yaml -- date > ticker.yaml
```

## ④ Production scenarios

**Batch đêm đúng giờ VN:** schedule cron đọc theo giờ của controller
nếu quên `timeZone` — job 2h sáng chạy lệch 7 giờ là lỗi kinh điển.
Đặt `timeZone` tường minh + `startingDeadlineSeconds` để controller
restart đúng lúc đến lịch thì bỏ lần đó thay vì chạy dồn.

**Chống chồng lịch:** ETL 10 phút nhưng thi thoảng kẹt 15 phút —
`concurrencyPolicy: Forbid` tránh hai bản chạy song song ghi đè dữ
liệu; muốn "bản mới thắng" thì `Replace`.

**Ngừng có kiểm soát:** trước upgrade cluster, patch `suspend: true` —
Job đang chạy vẫn kết thúc bình thường, chỉ không sinh Job mới; xong
đổi về `false`.

**Dọn dẹp tự động:** để mặc định, CronJob giữ 3 Job thành công + 1 Job
fail — chạy vài tuần là chất chồng object; giảm history limit và luôn
set `ttlSecondsAfterFinished` cho Job một lần.

## References

- [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
